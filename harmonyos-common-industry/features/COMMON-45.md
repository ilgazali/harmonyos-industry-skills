---
id: COMMON-45
title: Connection animation - a Canvas spinner driven by animateTo while a TCP socket connects
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/45_connect_animate.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/connect_animate-0000002508806245
sample: huawei_industry_tree/19_common_technical_solutions/downloads/ConnectAnimate.zip
kits: ["@kit.NetworkKit", "@kit.ArkUI", "@kit.ArkTS", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: ["socket.constructTCPSocketInstance", "TCPSocket.connect", "TCPSocket.close", "TCPSocket.on", "TCPSocket.off", "UIContext.animateTo", Canvas, CanvasRenderingContext2D, RenderingContextSettings, "context.arc", "context.stroke", ".rotate", "@ComponentV2", "@Local", "@ObservedV2", "@Trace", "@Monitor", setInterval, clearInterval, setTimeout, clearTimeout, "resourceManager.getStringSync"]
permissions: ["ohos.permission.INTERNET", "ohos.permission.GET_NETWORK_INFO"]
min_api: 20
modules: [entry]
findings: [HW-19-0143, HW-19-0144, HW-19-0145, HW-19-0146, HW-19-0147, HW-19-0148, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when a network operation takes long enough that the user needs to
see something happening - a connect, a handshake, a login - and the waiting state
must be animated rather than a static spinner.

The interesting content here is the **state plumbing**, not the drawing: how a
V2-observed model object drives the UI through `@Monitor`, how a looping
animation is built from `setInterval` plus `animateTo`, and how the click is
protected against repeated taps. The sample is also the industry's clearest
example of `@ComponentV2` / `@ObservedV2` / `@Trace` / `@Local` / `@Monitor` used
together.

## Feature checklist

- Model the connection state in an `@ObservedV2` class with `@Trace` fields.
- React to changes with `@Monitor('field.path')` in a `@ComponentV2` page.
- Loop an animation with `setInterval` + `animateTo`, and keep the animated value
  bounded (HW-19-0143).
- Register the socket's `on('message'|'connect'|'close')` handlers, and unregister
  them on close (HW-19-0144).
- Clear every timer in `aboutToDisappear` (HW-19-0145).
- Hold the throttled click handler on the component, not in `build()`
  (HW-19-0148).
- Declare only the permissions actually used (HW-19-0146).

## Architecture

Single-module project (`entry` HAP), one page:

| File (as shipped) | Documented as | Responsibility |
| --- | --- | --- |
| `pages/ButtonRoundPage.ets` | same | the whole UI, animation and timers |
| `Utils/NetworkManager.ets` | `util/` | `@ObservedV2` socket wrapper |
| `Mode/BindInfo.ets` | `model/` | address, port, family |
| `Mode/SocketInfo.ets` | `model/` | received-message shape |

Two of those directory names differ from the document (HW-19-0147).

**The state model is V2 throughout.** `NetworkManager` is `@ObservedV2` with two
`@Trace` booleans, `connected` and `connectedFail`. The page is `@ComponentV2`
holding it in an `@Local`, and reacts through two `@Monitor` methods rather than
by polling. This is the correct pairing: `@Trace` makes the field observable
*inside a class*, which is what lets `@Monitor('networkManager.connected')` watch
a path two levels deep - something V1's `@State` cannot do for a nested class
field.

**Two booleans, because failure is not the absence of success.** `connected` and
`connectedFail` are independent, so the page can distinguish "not yet connected"
from "tried and failed". `isConnectedFail` resets `connectedFail` to `false`
immediately after reacting, making it an edge signal rather than a state - which
is why it can fire again on the next failed attempt.

**The connection result is deliberately delayed.** `NetworkManager.connect`
wraps both outcomes in a 5-second `setTimeout` with the comment
`由于连接速度较快，为显示动画效果，延时处理` ("because the connection is fast, a
delay is added so the animation can be seen"). Do not carry this into production
code - but do notice that the timeout ids are stored and cleared at the top of
the next `connect`, which is the right shape for a debounced async result.

**The animation is three independent pieces.**

1. A `Canvas` draws a static 0.7-radian arc once, in `onReady`.
2. `.rotate({ angle: this.angle, ... })` turns it.
3. A 200 ms `setInterval` wraps each `angle` increment in `animateTo`, so every
   step is itself a 200 ms linear tween - interval period equals animation
   duration, which is what makes the motion continuous rather than stepped.

The angle is never reset because the reset tests for a value the increment skips
(HW-19-0143).

**A second interval animates the text.** `valueChange` appends dots to
`connectingText` every 800 ms, bounded by `connectText.length`.

**`centerX` / `centerY` are plain fields, not `@Local`.** They are assigned in
`onReady`, after the first build, so the `.rotate` center is `(0, 0)` on the
first frame and picks up the real center on the next rebuild - which the 200 ms
angle update supplies. It self-corrects, but the pattern is fragile: an animated
attribute reading an unobserved field.

## Implementation steps

1. **Model the connection** in an `@ObservedV2` class holding the `TCPSocket`
   instance and `@Trace` state flags.
2. **Connect**: `tcpSocket.connect({ address: { address, port, family }, timeout:
   5000 })`, setting the flags in `.then` and `.catch`.
3. **Register the listeners** - `on('message')`, `on('connect')`, `on('close')` -
   at connect time, and remove them in the close path (HW-19-0144).
4. **Watch from the page** with `@Monitor('networkManager.connected')` and
   `@Monitor('networkManager.connectedFail')`; stop the animation and show the
   result from there.
5. **Draw the arc** in `Canvas.onReady`, capture the center, and rotate the whole
   canvas by an `@Local` angle.
6. **Loop the animation** with `setInterval(..., 200)` calling `animateTo({
   duration: 200, curve: Curve.Linear })`, incrementing the angle modulo 360
   (HW-19-0143).
7. **Throttle the click** with a handler created once and stored on the component
   (HW-19-0148).
8. **Clean up** in `aboutToDisappear`: both intervals, and the socket
   (HW-19-0145).

## Verified snippets

All snippets below come from the sample project, not from the document.

### The observed model

`ConnectAnimate.zip#ConnectAnimate/entry/src/main/ets/Utils/NetworkManager.ets`

```ts
import { BusinessError } from '@kit.BasicServicesKit';
import { socket } from '@kit.NetworkKit';
import { BindInfo } from '../Mode/BindInfo';
import { SocketInfo } from '../Mode/SocketInfo';

@ObservedV2
export class NetworkManager {
  tcpSocket: socket.TCPSocket = socket.constructTCPSocketInstance();
  setTimeConnected: number = 0;
  setTimeConnectedFail: number = 0;
  @Trace connected: boolean = false; // 表示网络连接状态，false表示未连接/断开，true表示连接成功
  @Trace connectedFail: boolean = false; // 表示网络连接状态，false表示连接成功，true表示连接失败

  // 建立网络连接
  public connect(bindInfo: BindInfo) {
    clearTimeout(this.setTimeConnected);
    clearTimeout(this.setTimeConnectedFail);
    this.tcpSocket.connect({
      address: {
        address: bindInfo.address,
        port: bindInfo.port,
        family: bindInfo.family
      },
      timeout: 5000
    }).then(() => {
      // 由于连接速度较快，为显示动画效果，延时处理
      this.setTimeConnected = setTimeout(() => {
        this.connected = true;
        this.connectedFail = false;
        hilog.info(0x0001, 'NetworkManager', `connect success: ${this.connected}`);
      }, 5000);
    }).catch((err: BusinessError) => {
      // 由于连接速度较快，为显示动画效果，延时处理
      this.setTimeConnectedFail = setTimeout(() => {
        this.connected = false;
        this.connectedFail = true;
        hilog.error(0x0001, 'NetworkManager', `connect failed: ${JSON.stringify(err)}`);
      }, 5000);
    });
  }
```

Both branches clear the previous pair of timeouts before starting a new one, so a
second connect attempt cannot be overtaken by the first attempt's delayed result.

### The listeners that are never installed

`ConnectAnimate.zip#ConnectAnimate/entry/src/main/ets/Utils/NetworkManager.ets`

```ts
// 网络监听
public connecting() {                    // FIX (HW-19-0144): never called
  this.tcpSocket.on('message', (value: SocketInfo) => {
    let buffer = value.message;
    let dataView = new DataView(buffer);
    let str = '';
    for (let i = 0; i < dataView.byteLength; ++i) {
      str += String.fromCharCode(dataView.getUint8(i));
    }
    hilog.info(0x0001, 'NetworkManager', `on connect received: ${str}`);
  });
  this.tcpSocket.on('connect', () => { hilog.info(0x0001, 'NetworkManager', 'on connect'); });
  this.tcpSocket.on('close', () => { hilog.info(0x0001, 'NetworkManager', 'on close'); });
}

// 关闭网络连接，并关闭监听
public closeNetwork() {
  this.tcpSocket.close().then(() => {
    this.connected = false;
    this.tcpSocket.off('message');       // removing what was never added
    this.tcpSocket.off('connect');
    this.tcpSocket.off('close');
  }).catch((err: BusinessError) => {
    hilog.error(0x0001, 'NetworkManager', `close failed: ${JSON.stringify(err)}`);
  });
}
```

The `off` calls are correct and belong exactly where they are. The missing half
is a call to `connecting()`.

### Reacting to the model

`ConnectAnimate.zip#ConnectAnimate/entry/src/main/ets/pages/ButtonRoundPage.ets`

```ts
@Entry
@ComponentV2
struct ButtonRoundPage {
  @Local isStartConnect: boolean = false; // 是否点击连接按钮标记
  @Local networkManager: NetworkManager = new NetworkManager();
  @Local connectingText: string = '';
  @Local connectText: string = '连接网络';
  @Local angle: number = 0; // 旋转动画角度信息
  private bindNetInfo: BindInfo = new BindInfo(); // 连接的网络对象信息
  private settings: RenderingContextSettings = new RenderingContextSettings(true);
  private context: CanvasRenderingContext2D = new CanvasRenderingContext2D(this.settings);
  private centerX: number = 0; // 位置信息
  private centerY: number = 0; // 位置信息
  private resContext: Context = this.getUIContext().getHostContext() as Context;
  animateTimer: number | null = null; // 动画定时器
  valueChangeTimer: number | null = null; // 文字变化定时器

  aboutToAppear(): void {
    // 配置连接地址及接口
    this.bindNetInfo.address = this.resContext.resourceManager.getStringSync($r('app.string.address').id);
    this.bindNetInfo.port = 443; // 配置连接端口
    this.bindNetInfo.family = 1;
  }

  // 连接成功后关闭动画
  @Monitor('networkManager.connected')
  isConnected() {
    if (this.networkManager.connected) {
      this.isStartConnect = false;
      this.startAnimate();
      this.valueChange();
      this.getUIContext().getPromptAction().showToast({ message: '网络连接成功！' });
    } else {
      this.getUIContext().getPromptAction().showToast({ message: '网络已断开！' });
    }
  }

  // 连接失败后关闭动画
  @Monitor('networkManager.connectedFail')
  isConnectedFail() {
    if (this.networkManager.connectedFail) {
      this.isStartConnect = false;
      this.startAnimate();
      this.valueChange();
      this.networkManager.connectedFail = false;
      this.getUIContext().getPromptAction().showToast({ message: '网络连接失败！' });
    }
  }
```

Note the idiom in both monitors: set `isStartConnect = false` **first**, then call
`startAnimate()` and `valueChange()`. Those two methods branch on
`isStartConnect`, so calling them after clearing the flag is what takes their
`else` path and stops the timers. The methods are both starter and stopper.

The connect address comes from a string resource rather than a literal, and the
inline comment notes that from API 18 a domain name can be used instead of an IP.

### The looping animation

`ConnectAnimate.zip#ConnectAnimate/entry/src/main/ets/pages/ButtonRoundPage.ets`

```ts
// 连接动画，setInterval定时器重复执行达到循环效果。
startAnimate() {
  if (this.isStartConnect) {
    this.animateTimer = setInterval(() => {
      this.getUIContext()?.animateTo({
        duration: 200,
        curve: Curve.Linear,
        iterations: 1,
        playMode: PlayMode.Normal,
        onFinish: () => {
          hilog.info(0X0002, 'isPlaying', 'play end');
        }
      }, () => {
        if (this.angle === 360) {      // FIX (HW-19-0143): 50-step never hits 360
          this.angle = 0;
        }
        this.angle += 50;
      });
    }, 200);
  } else {
    this.angle = 0;
    clearInterval(this.animateTimer);
  }
}
```

The corrected increment:

```ts
}, () => {
  this.angle = (this.angle + 50) % 360;
});
```

### The Canvas arc

`ConnectAnimate.zip#ConnectAnimate/entry/src/main/ets/pages/ButtonRoundPage.ets`

```ts
// 环绕弧线动画
Canvas(this.context)
  .width(210)
  .height(210)
  .rotate({
    angle: this.angle,
    centerX: this.centerX,
    centerY: this.centerY
  })
  .visibility(this.isStartConnect ? Visibility.Visible : Visibility.Hidden) // 连接过程中，动画过程展示
  .onReady(() => {
    this.centerX = this.context.width / 2;
    this.centerY = this.context.height / 2;
    this.draw();
  });

// 绘制环绕动画
draw() {
  this.context.clearRect(0, 0, this.context.width, this.context.height);
  this.context.beginPath();
  this.context.arc(this.centerX, this.centerY, 100, 0, 0.7);
  this.context.lineWidth = 8;
  this.context.lineCap = 'round';
  this.context.strokeStyle = 'rgb(66, 129, 253)';
  this.context.stroke();
}
```

`draw()` runs once. The rotation is a component attribute, not a redraw - the
canvas content never changes, so the animation costs nothing per frame beyond the
transform.

### The text animation

`ConnectAnimate.zip#ConnectAnimate/entry/src/main/ets/pages/ButtonRoundPage.ets`

```ts
// 连接的中的文本变化
valueChange() {
  if (this.isStartConnect) {
    this.connectText = '连接中';
    this.connectingText = '';
    this.valueChangeTimer = setInterval(() => {
      if (this.connectingText.length < this.connectText.length) {
        this.connectingText += '.';
      } else {
        this.connectingText = '';
      }
    }, 800);
  } else {
    this.connectText = '连接网络';
    clearInterval(this.valueChangeTimer);
  }
}
```

The dot count is bounded by the length of `connectText` (three characters), so it
cycles `.`, `..`, `...`, empty.

### The throttled click

`ConnectAnimate.zip#ConnectAnimate/entry/src/main/ets/pages/ButtonRoundPage.ets`

```ts
// 三目运算判断是否进行防抖处理,点击连接时5000毫秒防抖，和网络连接超时时间一致。
.onClick(this.networkManager.connected ? () => {
  this.isStartConnect = !this.isStartConnect;
  if (this.isStartConnect && !this.networkManager.connected) {
    this.openConnect();
    this.startAnimate();
    this.valueChange();
  } else {
    this.isStartConnect = false;
    this.closeConnect();
  }
} : throttle(() => {                   // FIX (HW-19-0148): built inside build()
  /* same body */
}, 5000));

// 防抖函数，当开始连接时防抖，避免多次点击
export function throttle(func: (event: ClickEvent) => void, delay?: number) {
  let inThrottle: boolean;
  return (event: ClickEvent) => {
    if (!inThrottle) {
      func(event);
      inThrottle = true;
      setTimeout(() => inThrottle = false, delay ? delay : 1000);
    }
  };
}
```

The design intent is sound - throttle only the connect direction, since
disconnecting should be instant, and match the throttle window to the socket
timeout. The problem is where the wrapper is created.

## Permissions & config

`ConnectAnimate.zip#ConnectAnimate/entry/src/main/module.json5` declares **three**
permissions, one more than the document lists:

| Permission | Type | Used? |
| --- | --- | --- |
| `ohos.permission.INTERNET` | normal, install-time | yes - the TCP socket |
| `ohos.permission.GET_NETWORK_INFO` | normal, install-time | listed in the document |
| `ohos.permission.GET_WIFI_INFO` | normal, install-time | **no** - remove it (HW-19-0146) |

The `GET_WIFI_INFO` block carries `"abilities": ["com.samples.socket.EntryAbility"]`
while this bundle is `com.example.connectAnimate` - a copy from another project.

`build-profile.json5` pins `6.0.0(20)`. The connect address is a string resource
(`app.string.address`), read with `resourceManager.getStringSync`.

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later. A domain name may be used as the
  connect address from API 18.
- **`@Trace` is required** for a class field to be observable from a
  `@ComponentV2` page; `@ObservedV2` on the class alone is not enough.
- **`@Monitor` watches a path**, so the whole chain from the `@Local` field down
  must be observable.
- **`animateTo` needs a `UIContext`.** The sample uses
  `this.getUIContext()?.animateTo(...)` rather than the global function.
- **`Canvas.onReady` is the first point at which `context.width`/`height` are
  known** - the center cannot be computed before it.
- **Interval period should match the animation duration** for a continuous loop;
  a shorter period queues overlapping tweens.
- **`family: 1`** selects IPv4 in `NetAddress`; `BindInfo`'s own default is `2`,
  and `aboutToAppear` overrides it.
- **Devices.** Per `module.json5`.

## Pitfalls

- **`if (this.angle === 360)` never fires, which is incorrect** - incrementing by
  50 from 0 skips 360 entirely, so the angle grows without bound for as long as
  the animation runs. Use `(this.angle + 50) % 360`. (HW-19-0143)
- **`connecting()` is never called, which is incorrect** - no socket listener is
  ever registered, so nothing can be received, while `closeNetwork` unregisters
  three handlers that were never added. (HW-19-0144)
- **The page has no `aboutToDisappear`, which is incorrect** - leaving mid-connect
  leaves a 200 ms interval, an 800 ms interval and two 5-second timeouts running
  against a destroyed component. (HW-19-0145)
- **`GET_WIFI_INFO` is requested but never used, is absent from the document, and
  its `usedScene` names an ability from `com.samples.socket` - all incorrect.**
  Remove the block. (HW-19-0146)
- **The documented tree says `model/` and `util/`; the ZIP ships `Mode/` and
  `Utils/`, which is incorrect** - and `Mode` is a misspelling of `Model`.
  (HW-19-0147)
- **`throttle(...)` is called inside `build()`, which is incorrect** - the guard
  state belongs to a build-time closure rather than to the component, so
  re-evaluating the attribute discards it. Store the wrapper in a field.
  (HW-19-0148)
- **Do not keep the 5-second artificial delay.** It exists so the animation is
  visible in the demo and would be a five-second lie in a real product.
- **Do not read unobserved fields from animated attributes.** `centerX`/`centerY`
  are plain fields consumed by `.rotate`; it works here only because the angle
  update forces a rebuild moments later.

## References

- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-socket -
  `constructTCPSocketInstance`, `TCPSocket.connect`, `NetAddress`, `on`/`off`.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/arkts-attribute-animation-apis -
  `animateTo` and its `AnimateParam`.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-components-canvas-canvas -
  `Canvas`, `onReady`, `CanvasRenderingContext2D.arc`.
- `documentation/harmonyos-references/05_common-capabilities/js-apis-timer.md` -
  `setInterval` / `clearInterval` / `setTimeout` / `clearTimeout`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-timer
- `documentation/harmonyos-guides/03_application-framework/arkts-page-custom-components-lifecycle.md` -
  `aboutToAppear` / `aboutToDisappear`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-page-custom-components-lifecycle
- https://developer.huawei.com/consumer/cn/doc/harmonyos-faqs/faqs-arkts-162 -
  intercepting repeated rapid clicks, the reference the document cites for the
  throttle.
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/connect_animate-0000002508806245
