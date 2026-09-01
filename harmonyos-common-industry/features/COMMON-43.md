---
id: COMMON-43
title: Custom common events - inter-process communication between two applications
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/43_customizing_common_event.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/customizing_common_event-0000002470917738
sample: huawei_industry_tree/19_common_technical_solutions/downloads/CustomizingCommonEvent.zip
kits: ["@kit.BasicServicesKit", "@kit.ArkUI", "@kit.PerformanceAnalysisKit"]
apis: ["commonEventManager.publish", "commonEventManager.createSubscriber", "commonEventManager.subscribe", "commonEventManager.unsubscribe", CommonEventSubscribeInfo, CommonEventPublishData, CommonEventSubscriber, CommonEventData, "@CustomDialog", CustomDialogController, "@Provide", aboutToAppear, aboutToDisappear]
permissions: []
min_api: 20
modules: [CommonEventSender/entry, CommonEventReceiver/entry]
findings: [HW-19-0132, HW-19-0133, HW-19-0134, HW-19-0135, HW-19-0136, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when two **separate applications** - separate bundles, separate
processes - must notify each other of something, and neither one needs a reply.
A payment app telling a messaging app that a purchase completed is the shape the
sample demonstrates.

Common events are a broadcast: the publisher names an event string and hands over
a payload, the system delivers it to whatever has subscribed. There is no
handshake, no return channel, and no delivery guarantee visible to the sender
beyond the publish callback saying the event was accepted for delivery.

If the two ends are in the same application, this is the wrong tool - use
`@Provide`/`@Consume`, an `emitter`, or a shared store.

## Feature checklist

Publisher:

- `commonEventManager.publish(event, options, callback)` with the receiver's
  bundle name in `bundleName` and the payload serialized into `data`.
- Commit UI state **inside** the callback, only on success (HW-19-0135).

Subscriber:

- Build a `CommonEventSubscribeInfo` with the same `events` string and the
  publisher's `publisherBundleName`.
- `createSubscriber` → store the subscriber → `subscribe`.
- **`unsubscribe` in `aboutToDisappear`** - the step the document omits
  (HW-19-0133) and the sample gets wrong (HW-19-0132).

## Architecture

Two independent projects inside one ZIP:

| Project | Bundle | Role |
| --- | --- | --- |
| `CommonEventSender` | `com.example.commoneventsender` | product-detail page with a Buy button; publishes `purchaseEvent` |
| `CommonEventReceiver` | `com.example.commoneventreceiver` | chat-style app; subscribes and prepends a message |

Both must be installed for the demo to do anything.

**The pairing is by two strings, declared on both sides.** The event name
`'purchaseEvent'` and the counterparty's bundle name. The publisher sets
`bundleName: 'com.example.commoneventreceiver'` in the publish data, which
restricts delivery to that bundle; the subscriber sets
`publisherBundleName: 'com.example.commoneventsender'`, which restricts what it
will accept. Get either string wrong and the event silently goes nowhere - there
is no error, because a common event with no matching subscriber is not a failure.

**The payload is a JSON string.** `CommonEventPublishData.data` is a string, so
the sender does `JSON.stringify(eventData)` and the receiver does
`JSON.parse(data.data)` inside a `try/catch`. Both ends declare their own
`ProductInfo` interface - the receiver redeclares it locally at the top of its
`MainPage.ets` - because there is no shared type across bundles. Nothing checks
that the two shapes agree; the `try/catch` around the parse is the only guard,
and it catches malformed JSON, not a mismatched shape.

**Note what the payload contains and what survives.** The sender puts
`picture: $r('app.media.picture')` into the object it stringifies. A `Resource`
serializes to its numeric id and module metadata, which is meaningless in another
bundle. The receiver reads only `purchaseData.price` and ignores it, so nothing
breaks - but a resource reference must never be treated as transferable across
applications.

**The subscriber's lifetime is not the page's lifetime.** `createSubscriber` and
`subscribe` register with a system service. The page holds a reference; the
service holds the callback. Destroying the page releases neither. This is the
whole reason `unsubscribe` matters, and it is what both the document and the
sample fail to deliver.

## Implementation steps

**Publisher:**

1. Build the payload and serialize it: `data: JSON.stringify(eventData)`.
2. Call `publish` with the receiver's bundle name:
   ```ts
   commonEventManager.publish('purchaseEvent', {
     bundleName: 'com.example.commoneventreceiver',
     data: JSON.stringify(eventData),
   }, (err: BusinessError) => { /* ... */ })
   ```
3. **In the callback**: on `err`, log it with a matching format specifier
   (HW-19-0134) and tell the user; on success, commit the UI state - open the
   confirmation, disable the button (HW-19-0135).

**Subscriber:**

4. Declare the subscription in `aboutToAppear`:
   ```ts
   let subscriberInfo: commonEventManager.CommonEventSubscribeInfo = {
     events: ['purchaseEvent'],
     publisherBundleName: 'com.example.commoneventsender'
   }
   ```
5. `createSubscriber(subscriberInfo, (err, subscriber) => { ... })`, store the
   subscriber in a field, then `subscribe(this.subscriber, (err, data) => ...)`.
6. Parse inside `try/catch` and guard `data.data` before parsing.
7. **`aboutToDisappear()`** - exactly that name - calls
   `commonEventManager.unsubscribe(this.subscriber)` in a `try/catch` and clears
   the field (HW-19-0132).

## Verified snippets

All snippets below come from the sample project, not from the document.

### Publishing

`CustomizingCommonEvent.zip#CustomizingCommonEvent/CommonEventSender/entry/src/main/ets/pages/MainPage.ets`

```ts
import { ProductInfo } from '../model/ProductInfoModel';
import { BusinessError, commonEventManager } from '@kit.BasicServicesKit';

Button('立即购买')
  .width(CommonConstants.PURCHASE_BUTTON_WIDTH)
  .height(CommonConstants.PURCHASE_BUTTON_HEIGHT)
  .fontSize(CommonConstants.NORMAL_FONTSIZE)
  .fontWeight(FontWeight.Medium)
  .fontColor(Color.White)
  .margin({ top: CommonConstants.MARGIN_EIGHT, right: CommonConstants.MARGIN_THIRTY_TWO })
  .enabled(!this.isPurchased)
  .onClick(() => {
    this.dialogController.open()      // FIX (HW-19-0135): move into the success path
    this.isPurchased = true           // FIX (HW-19-0135): move into the success path
    let eventData: ProductInfo = {
      picture: $r('app.media.picture'),
      title: '西班牙菲力牛排',
      description: '西班牙进口牛排，口味新鲜',
      price: 299,
    }
    commonEventManager.publish('purchaseEvent', {
      bundleName: 'com.example.commoneventreceiver',
      data: JSON.stringify(eventData),
    }, (err: BusinessError) => {
      if (err) {
        // FIX (HW-19-0134): no specifier, err.code is dropped
        hilog.error(0xFF00,'commonEventSender','事件发布失败:', err.code)
      }
    })
  })
```

The corrected ordering:

```ts
.onClick(() => {
  const eventData: ProductInfo = { /* ... */ };
  commonEventManager.publish('purchaseEvent', {
    bundleName: 'com.example.commoneventreceiver',
    data: JSON.stringify(eventData),
  }, (err: BusinessError) => {
    if (err) {
      hilog.error(0xFF00, 'commonEventSender', '事件发布失败: %{public}d', err.code);
      return;                       // button stays enabled, no success dialog
    }
    this.isPurchased = true;
    this.dialogController.open();
  });
})
```

### The confirmation dialog

`CustomizingCommonEvent.zip#CustomizingCommonEvent/CommonEventSender/entry/src/main/ets/pages/MainPage.ets`

```ts
@CustomDialog
struct CustomDialogExample {
  controller: CustomDialogController

  build() {
    Column() {
      Text('已支付成功')
        .fontColor($r('app.color.option_text'))
        .fontWeight(FontWeight.Bold)
        .fontSize(CommonConstants.TITLE_FONTSIZE)
        .height(56)
      Text('恭喜你，已支付成功')
        .fontColor($r('app.color.option_text'))
        .fontWeight(FontWeight.Medium)
        .fontSize(CommonConstants.NORMAL_FONTSIZE)
      Text('我知道了')
        .fontColor($r('app.color.blue'))
        .fontWeight(FontWeight.Medium)
        .fontSize(CommonConstants.NORMAL_FONTSIZE)
        .height(56)
        .onClick(() => {
          this.controller.close()
        })
    }
    .height(144)
    .justifyContent(FlexAlign.Center)
  }
}

// in the page:
dialogController: CustomDialogController = new CustomDialogController({
  builder: CustomDialogExample(),   // 关联弹窗组件
  isModal: true                     // FIX (HW-19-0136): comment says non-modal
})
```

### Subscribing

`CustomizingCommonEvent.zip#CustomizingCommonEvent/CommonEventReceiver/entry/src/main/ets/pages/MainPage.ets`

```ts
import { BusinessError, commonEventManager } from '@kit.BasicServicesKit';

interface ProductInfo {
  picture?: Resource,
  title: ResourceStr,
  description: ResourceStr,
  price: number,
}

@Entry
@Component
struct HomePage {
  @State currentPageIndex: number = Constants.CHAT_INDEX
  @Provide sessionList: MessageItem[] = []
  private subscriber: commonEventManager.CommonEventSubscriber | null = null

  aboutToAppear() {
    let subscriberInfo: commonEventManager.CommonEventSubscribeInfo = {
      events: ['purchaseEvent'],                        // 与发送方一致的自定义事件名
      publisherBundleName: 'com.example.commoneventsender'  // 发送方包名
    }

    commonEventManager.createSubscriber(subscriberInfo,
      (err: BusinessError, subscriber: commonEventManager.CommonEventSubscriber) => {
        if (err) {
          return
        }
        this.subscriber = subscriber
        commonEventManager.subscribe(this.subscriber, (err, data) => {
          if (err || !data) {
            return
          }
          try {
            if (data.data) {
              let purchaseData: ProductInfo = JSON.parse(data.data)
              this.sessionList.unshift({
                id: 0,
                receiver: Constants.PURCHASE_INFO,
                avatar: $r('app.media.puchase_message'),
                content: Constants.DEDUCTED_MESSAGE + purchaseData.price,
                sendingTime: Constants.SENDING_TIME
              })
            }
          } catch (e) {
            hilog.error(0xFF00,'commonEventSender','数据解析失败')   // FIX (HW-19-0136): wrong tag
          }
        })
      })
  }
```

### The cleanup that never runs

`CustomizingCommonEvent.zip#CustomizingCommonEvent/CommonEventReceiver/entry/src/main/ets/pages/MainPage.ets`

```ts
disappear() {                       // FIX (HW-19-0132): must be aboutToDisappear()
  if (this.subscriber) {
    try {
      commonEventManager.unsubscribe(this.subscriber)
      this.subscriber = null // 释放引用
    } catch (error) {
      hilog.error(0xFF00, 'TAG', '取消订阅失败: ' + error.message)
    }
  }
}
```

The body is correct - the guard, the `try/catch`, the reference release. Only the
method name is wrong, and that is enough to make all of it dead code. Rename to
`aboutToDisappear()` and nothing else changes.

## Permissions & config

**No permissions are declared in either project**, and none are required: a
custom common event published to a named bundle needs no permission on either
side. This is specific to custom events - many **system** common events are
permission-gated, and `CommonEventPublishData` also supports a
`subscriberPermissions` array for restricting who may receive an event, which
neither project uses.

Both `build-profile.json5` files pin `6.0.0(20)` for `targetSdkVersion` and
`compatibleSdkVersion`. Both `module.json5` files declare `deviceTypes: ["phone",
"tablet", "2in1"]`, a single `EntryAbility` with the home skill, and an
`EntryBackupAbility`.

The bundle names are the contract: `AppScope/app.json5` declares
`com.example.commoneventsender` and `com.example.commoneventreceiver`
respectively, and both strings are hard-coded in the opposite project's source.

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later. `publish`, `createSubscriber`,
  `subscribe` and `unsubscribe` are all long-standing APIs; the module's atomic
  service support is API 11+.
- **Both applications must be installed.** There is no error when no subscriber
  matches - `publish` succeeds and the event is simply not delivered.
- **The payload is a string.** Serialize and parse it yourself, and guard the
  parse.
- **`Resource` values do not cross bundles.** They serialize to ids that mean
  nothing in the receiving application.
- **The subscriber outlives its component** unless explicitly unsubscribed.
- **Renaming either bundle breaks the pairing silently** - the strings appear in
  the opposite project's source with no compile-time link.
- **Devices.** phone, tablet, 2in1 per both `module.json5` files.

## Pitfalls

- **The receiver's cleanup is named `disappear()`, which is incorrect;** the
  framework calls `aboutToDisappear()`, so the `unsubscribe` never executes and
  every page creation adds another live subscriber that replays the handler.
  (HW-19-0132)
- **The document's receiver instructions stop at `subscribe`, which is
  incomplete;** the module reference lists unsubscribe as one of its three
  capabilities, and omitting the step is what let the sample's dead cleanup pass
  unnoticed. (HW-19-0133)
- **`hilog.error(..., '事件发布失败:', err.code)` prints no code, which is
  incorrect** - hilog requires "the number and type of parameters must map to the
  identifier in the format string". Use `'事件发布失败: %{public}d'`.
  (HW-19-0134)
- **The success dialog opens and the button is disabled before `publish`
  reports, which is incorrect** - a failed publish leaves the user told that
  payment succeeded, with no retry path. Commit UI state in the callback.
  (HW-19-0135)
- **`isModal: true` is annotated "设为非模态弹窗", which is incorrect** - `true`
  is modal and is also the default; and the receiver logs its parse failures
  under `commonEventSender`. (HW-19-0136)
- **Do not treat a matching event name as sufficient.** Without
  `publisherBundleName` on the subscriber, any application that knows the string
  can publish an event your code treats as a completed purchase. The sample sets
  it; keep it.
- **Do not assume the payload's shape.** `JSON.parse` returns whatever arrived;
  the `as ProductInfo` cast is a compile-time claim, not a runtime check.

## References

- `documentation/harmonyos-references/03_system/js-apis-commoneventmanager.md` -
  `publish`, `createSubscriber`, `subscribe`, `unsubscribe`,
  `CommonEventSubscribeInfo`, `CommonEventPublishData`.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-commoneventmanager
- `documentation/harmonyos-guides/03_application-framework/arkts-page-custom-components-lifecycle.md` -
  `aboutToAppear` / `aboutToDisappear`, the names the framework calls.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/arkts-page-custom-components-lifecycle
- `documentation/harmonyos-references/03_system/js-apis-hilog.md` - format string
  and argument correspondence.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-hilog
- `documentation/harmonyos-references/02_application-framework/ts-methods-custom-dialog-box.md` -
  `CustomDialogController` and `isModal`.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-methods-custom-dialog-box
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/customizing_common_event-0000002470917738
