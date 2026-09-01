---
id: SOCIAL-20
title: Weak-network chat - heartbeat, exponential backoff, and a reconnect path that never fires
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/20_weak_network_reconnection.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/weak_network_reconnection-0000002288904746
sample: huawei_industry_tree/14_social_communication/downloads/LoadingChat.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.NetworkKit", "@kit.PerformanceAnalysisKit"]
apis: [base, connection, emitter, hilog, util, webSocket, window]
permissions: [ohos.permission.INTERNET, ohos.permission.GET_NETWORK_INFO]
min_api: 20
modules: [entry (entry)]
findings: [HW-14-0002, HW-14-0044, HW-14-0045, HW-14-0046, HW-14-0047, HW-14-0087, HW-14-0088]
status: verified-with-fixes
---

## When to use

Load this card when a **long-lived socket has to survive a bad network** -
instant messaging, live comments, presence, order tracking, a collaborative
cursor. The user is on a train; the connection drops every ninety seconds; the
app must not look broken and must not lose messages.

The sample assembles the four pieces such a client needs: an application-level
heartbeat (`ping`/`pong` on a 25-second interval with a 3-second grace), a
reconnect with exponential backoff and a retry cap, a system network-state
subscription so recovery is event-driven rather than timer-driven, and an
optimistic UI where a message appears immediately with a spinner and the
spinner clears when the server acknowledges it. That fourth piece is the one
users actually see, and it is the piece this sample gets right.

**The other three are broken in ways that matter, and the card is written
around the corrections.** `HW-14-0044` and `HW-14-0045` are both high: the
advertised network-recovery reconnect is unreachable code, and the reconnect
machinery that *does* run degenerates into a loop that tears down every
healthy connection it makes. Use the structure; use the corrected snippets.

## Feature checklist

- On entering the chat page, start a `NetConnection` subscription and open a
  WebSocket to the chat endpoint.
- Typing enables the Send button; sending appends the message to the list
  immediately with a spinner over it, and the spinner for that message id
  clears when the server echoes it back.
- The list auto-scrolls to the bottom whenever the data changes.
- A heartbeat `ping` goes out every 25 s; a `pong` within 28 s counts as alive.
- A missed heartbeat closes the connection and schedules a reconnect with
  exponential backoff (1 s, 2 s, 4 s, 8 s, 16 s), capped at 5 attempts and 30 s.
- Messages that failed to send are cached and re-sent when the socket reopens.
- When the system reports the network is available again, reconnect.
  (**Unreachable in the sample** - see `HW-14-0044`.)
- Leaving the page closes the socket and removes every emitter listener.

## Architecture

One `entry` module. A page, a send view, a data model, and two singletons that
talk to each other only through `emitter`.

```
entry/src/main/ets
├── entryability/EntryAbility.ets   full-screen window, avoid areas -> AppStorage
├── entrybackupability/
├── model/DataModel.ets      @ObservedV2 MsgContent { msgID, @Trace isSuccessed, content }
├── pages/Chat.ets           @Entry: the message list, the emitter subscriptions, the lifecycle
├── utils
│  ├── NetworkMonitor.ets    connection.createNetConnection -> emitter('networkStateChange')
│  └── WebSocketManager.ets  the socket, the heartbeat, the backoff, the resend cache
└── view/MessageSend.ets     RichEditor composer + optimistic send
```

The documented tree matches the zip.

**The design decision worth copying** is that the two utilities are decoupled
singletons joined by the system event emitter, not by imports of each other:

```typescript
// NetworkMonitor publishes
netConnection.on('netAvailable', () => {
  emitter.emit('networkStateChange', { data: { 'content': 'available', 'id': 'networkStateChange' } });
});

// WebSocketManager subscribes, in its constructor
emitter.on('networkStateChange', (eventData: emitter.EventData) => { /* ... */ });

// Chat subscribes to the socket's own events
emitter.on('websocketOpen', ...); emitter.on('websocketMessage', ...);
emitter.on('websocketClose', ...); emitter.on('websocketError', ...);
```

Three participants, no direct references, one bus. The socket manager does not
know a UI exists, the network monitor does not know a socket exists, and the
page can be replaced without touching either. That is the right architecture
for this problem and it is why the corrections below are all local to one
file.

**The design decision worth avoiding** is that `WebSocketManager` has no
variable saying whether it is connected. Its state is spread over `ws`,
`retryCount`, `retryTimer`, `heartbeatTimer` and `isManualClose`, and the code
tries to infer "am I connected?" from whether `ws` is null. It never is, after
the first `connect()`. Both high-severity findings are that inference failing:
one path never fires because `!this.ws` is never true, the other loops because
nothing distinguishes "the socket I just replaced closed" from "my current
socket closed".

## Implementation steps

1. **Declare `ohos.permission.INTERNET` and `ohos.permission.GET_NETWORK_INFO`.**
   Both are `system_grant`; no runtime request is needed.
2. **Use a `wss://` endpoint** (`HW-14-0002`). The doc's
   `ws://echo.websocket.org` is cleartext and, as of today, the public echo
   service no longer accepts plain `ws`, so the sample fails functionally as
   well as on principle.
3. **Track connection state explicitly.** One boolean set in `handleOpen` and
   cleared in `handleClose`/`handleError` - not `!this.ws` (`HW-14-0044`), and
   reset `retryCount` when the network comes back, otherwise five failures
   during an outage permanently disable reconnection (`HW-14-0044`).
4. **Detach the old socket's listeners before closing it,** so the socket you
   are replacing cannot schedule a reconnect against the socket that replaces
   it, and **cancel the pending retry timer in `handleOpen`** so a successful
   open disarms every retry scheduled while it was in flight (`HW-14-0045`).
5. **Drain the resend queue by copy-and-clear,** never by mutating the array
   you are iterating (`HW-14-0046`).
6. **`return` after handling a control frame** so `ping`/`pong` never reach the
   chat-message parser (`HW-14-0047`).
7. **Render optimistically**: push the message with `isSuccessed = false`,
   show a spinner, and let the echo flip the flag by `msgID`.
8. **Close the socket and `emitter.off` every event in `aboutToDisappear`.**

## Verified snippets

All snippets are from `LoadingChat.zip`. Corrected forms are marked.

**Network recovery — `entry/src/main/ets/utils/NetworkMonitor.ets` and `WebSocketManager.ets`** (corrected, see `HW-14-0044`)

```typescript
// NetworkMonitor.ets - as shipped, this half is correct
startMonitor(): void {
  if (this.isListening) {
    return;
  }
  let netConnection: connection.NetConnection = connection.createNetConnection();
  netConnection.register((error: BusinessError) => {
    hilog.info(0x0000, 'ccTest', JSON.stringify(error));
  });
  netConnection.on('netAvailable', () => {
    let eventData: emitter.EventData = {
      data: { 'content': 'available', 'id': 'networkStateChange' }
    };
    emitter.emit('networkStateChange', eventData);
  });
  // netUnavailable is published the same way, with 'unavailable'
  this.isListening = true;
}

// WebSocketManager.ets
private isConnected = false;                   // FIX: the state the sample never keeps

constructor() {
  emitter.on('networkStateChange', (eventData: emitter.EventData) => {
    if (eventData.data!.content === 'available' && !this.isConnected) {  // FIX: was !this.ws
      this.retryCount = 0;                     // FIX: absent - the cap never released
      this.reconnect();
    }
  });
}
```

**`register()` before `on()` is not optional** - the reference is explicit
that events only arrive after `register`, and the `isListening` guard makes
`startMonitor` idempotent so a page revisit does not stack subscriptions.
That much the sample gets right.

The subscriber does not. `this.ws` is assigned in `ac_connect` and set to
`null` in exactly one place: the manual `close()`, which also sets
`isManualClose = true` and thereby blocks `reconnect()` anyway. After a
network drop, `ws` still holds the stale socket object, so `!this.ws` is
`false` and the branch the document advertises - 当检测到网络可用且WebSocket
连接断开后，重新连接服务端 ("when the network is detected as available and the
WebSocket connection is down, reconnect to the server") - never executes.
An explicit `isConnected`, set true in `handleOpen` and false in
`handleClose`/`handleError`, is the whole fix. Resetting `retryCount` on
recovery is the second half: the backoff caps at five attempts, so a tunnel
longer than 1+2+4+8+16 = 31 seconds exhausts them and `_scheduleReconnect`
returns early forever.

**The connect/close churn loop — `WebSocketManager.ets`** (corrected, see `HW-14-0045`)

```typescript
private ac_connect(): void {
  if (this.ws) {
    this.ws.off('open');                 // FIX: detach before closing, so the socket we are
    this.ws.off('close');                //      replacing cannot schedule a reconnect against
    this.ws.off('message');              //      the socket that replaces it
    this.ws.off('error');
    this.ws.close();
  }

  try {
    this.ws = webSocket.createWebSocket();
    this.ws.on('open', (err: BusinessError, value: Object) => {
      if (err) {
        hilog.info(0x0000, 'ccTest', 'webSocket open 报错：' + JSON.stringify(err));
        return;
      }
      this._flushQueue();
      this.handleOpen();
    });
    this.ws.on('close', (err, value) => { if (!err) { this.handleClose(); } });
    this.ws.on('message', (err, value) => { if (!err) { this.handleMessage(value); } });
    this.ws.on('error', (err) => { this.handleError(err); });
    this.ws.connect(this.url, this.options).catch((err: BusinessError) => {
      hilog.info(0x0000, 'ccTest', 'webSocket 连接 报错：' + JSON.stringify(err));
    });
  } catch (err) {
    hilog.error(0x0000, 'ccTest', JSON.stringify(err));
    this._scheduleReconnect();
  }
}

private handleOpen(): void {
  this.isConnected = true;               // FIX
  this.retryCount = 0;
  if (this.retryTimer) {                 // FIX: a successful open must disarm every retry
    clearTimeout(this.retryTimer);       //      that was scheduled while it was in flight
    this.retryTimer = null;
  }
  this._startHeartbeat();
  emitter.emit('websocketOpen');
}

private handleClose(): void {
  this.isConnected = false;              // FIX
  this._clearTimers();
  if (!this.isManualClose) {
    this._scheduleReconnect();
  }
  emitter.emit('websocketClose');
}
```

**Follow the shipped code through one heartbeat timeout and the loop is
obvious.** `_startHeartbeat` sees no `pong` in 28 seconds and calls
`handleClose`, which schedules a reconnect. The timer fires, `reconnect()`
calls `ac_connect`, which closes the old socket **and then immediately creates
the new one**. The old socket's `close` event arrives *after* the new socket
exists and runs the same `handleClose`, scheduling another retry. The new
socket opens a moment later; `handleOpen` resets `retryCount` to 0 but leaves
that pending timer armed. One to two seconds later the timer fires,
`ac_connect` closes the *healthy* socket, and the cycle repeats - forever. The
retry cap never engages precisely because every successful open resets the
counter.

Two lines break it. Detaching the old socket's handlers before `close()` stops
a socket you have abandoned from speaking for the manager. Clearing
`retryTimer` inside `handleOpen` makes "we are connected" cancel "we intend to
reconnect" - and note that `_clearTimers()` in `handleClose` runs *before*
`_scheduleReconnect()` arms the new one, so it cannot substitute. The `try`
block is also worth reading for what it does not catch: `ws.connect()`'s
rejection is only logged, and on some paths produces no `close` and no `error`
event, so nothing schedules a retry.

**The resend cache and the control frames — `WebSocketManager.ets`** (corrected, see `HW-14-0046`, `HW-14-0047`)

```typescript
// FIX: sample inlines this in the 'open' handler as
//   this.loadingData.forEach((value, index) => { this.ws?.send(value)...;
//                                                this.loadingData.slice(index, index); });
// `if (this.loadingData)` is always true, `slice` returns a new array and mutates nothing,
// and splicing during forEach would skip elements anyway.
private _flushQueue(): void {
  const pending = this.loadingData;
  this.loadingData = [];                       // clear first: a failed send re-queues itself
  pending.forEach((value: string) => {
    this.ws?.send(value).catch((err: BusinessError) => {
      this.loadingData.push(value);
      hilog.error(0x0000, 'ccTest', 'WebSocket send error:' + JSON.stringify(err));
    });
  });
}

private handleMessage(data: string | ArrayBuffer): void {
  // 此处为发送'ping'后接收到'pong'消息的设置
  if (typeof data === 'string' && data === 'pong') {
    this.lastPongTime = Date.now();
    return;                                    // FIX: absent - control frames fell through
  }
  // 此处为接收到服务器发送的'ping'消息时，返回'pong'
  if (typeof data === 'string' && data === 'ping') {
    this.ws?.send('pong').catch((err: BusinessError) => {   // FIX: sample drops the promise
      hilog.error(0x0000, 'ccTest', 'Send pong error:' + JSON.stringify(err));
    });
    return;                                    // FIX
  }
  let msgID: string = data.toString().substring(0, 36);
  let content: string = data.toString().substring(37, data.toString().length - 1);
  let eventData: emitter.EventData = {
    data: { 'content': content, 'id': 'websocketMessage', 'msgID': msgID }
  };
  emitter.emit('websocketMessage', eventData);
}
```

**`slice(index, index)` is a no-op where `splice(index, 1)` was meant**, so
the offline queue never drains: every message that once failed to send is
re-sent on *every* subsequent reconnect, and the duplicates compound with each
reconnection. Copy-and-clear is the safe drain - it also removes the
mutate-while-iterating hazard that a naive `splice` fix would introduce, and
it lets a send that fails again re-queue itself into the now-empty array
without being re-visited in this pass.

**The message parser is positional and has no envelope.** `msgID` is
`substring(0, 36)` because `util.generateRandomUUID(false)` is exactly 36
characters, and the content is everything from index 37 to one before the end
- offset 37 rather than 36 and the trailing trim exist because `MessageSend`
concatenates `msgID + JSON.stringify(content)`, and the JSON quotes are being
stripped by arithmetic. Without the `return` statements, `'ping'` and `'pong'`
run through this same parser and are emitted as `websocketMessage` events with
a five-character `msgID` and empty content. `Chat.ets` happens to ignore them
because no message has that id; any second consumer would process garbage.
A one-byte JSON envelope with a `type` field would have removed all of this.

**Optimistic send and the spinner — `Chat.ets` / `MessageSend.ets`** (as shipped, endpoint per `HW-14-0002`)

```typescript
// Chat.ets
aboutToAppear() {
  networkMonitor.startMonitor();
  webSocketManager.connect('ws://echo.websocket.org');   // HW-14-0002: must be wss://

  emitter.on('websocketMessage', (data) => {
    setTimeout(() => {
      this.data.map((item) => {
        if (item.msgID.toString() === data?.data?.msgID) {
          item.isSuccessed = true;                       // @Trace - flips the one spinner
        }
      });
    }, 2500);                                            // artificial latency, demo only
  });
  this.getUIContext().setKeyboardAvoidMode(KeyboardAvoidMode.RESIZE);
}

aboutToDisappear() {
  webSocketManager.close();
  emitter.off('websocketOpen');
  emitter.off('websocketMessage');
  emitter.off('websocketClose');
  emitter.off('websocketError');
}

// MessageSend.ets
private async sendMessage() {
  let message: MsgTextImage[] = [];
  richController.getSpans().forEach(span => {
    message.push({ content: (span as RichEditorTextSpanResult).value.toString() });
  });
  if (message.length > 0) {
    let msgID: string = util.generateRandomUUID(false);
    let content = JSON.stringify(message[0].content);
    let msgContent: MsgContent = new MsgContent(msgID, false, content.substring(1, content.length - 1));
    this.data.push(msgContent);                          // appears immediately, unacknowledged
    let sendMessage = msgID + content;
    webSocketManager.send(sendMessage);
  }
  richController.deleteSpans();
}
```

**This is the part of the sample to copy verbatim.** The message is appended
with `isSuccessed = false` before the socket is touched, so the UI never waits
on the network. `MsgContent` is `@ObservedV2` with `@Trace isSuccessed`, which
means flipping that one field on one element re-renders that element's spinner
and nothing else - no list-wide invalidation. The row's
`.visibility(item.isSuccessed ? Visibility.Hidden : Visibility.Visible)` on
the loading image is the entire "pending" affordance. The `2500` ms
`setTimeout` is the sample simulating a slow server against an echo endpoint
that answers instantly; delete it against a real backend.

`aboutToDisappear` is a good example of what the rest of the manager should
look like: it closes the socket and removes all four subscriptions. Note the
asymmetry - `NetworkMonitor` has no `stopMonitor`, so the `NetConnection`
registered in `aboutToAppear` is never unregistered.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.INTERNET" },
  { "name": "ohos.permission.GET_NETWORK_INFO" }
]
```

- Both are `system_grant`: declaring them is sufficient, and no
  `requestPermissionsFromUser` call is needed. Neither carries `reason` or
  `usedScene`, which is correct for system-grant permissions.
- `INTERNET` is what the WebSocket needs; `GET_NETWORK_INFO` is what
  `connection.createNetConnection().register()` needs. Omitting the second one
  makes the recovery path silently dead - which, in this sample, it already
  is for a different reason.
- Transport security is not a manifest switch here; the control is the URL
  scheme, so use `wss://` (`HW-14-0002`).

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The endpoint is an echo server**, so "the server acknowledged my message"
  and "the server sent me a message" are the same event. A real protocol needs
  a distinct ack frame; the `substring(0, 36)` id parsing is not a wire format.
- `MAX_RETRY_COUNT` is 5 and `MAX_RETRY_DELAY` 30 s, but the backoff formula
  `1000 * 2^retryCount` reaches only 16 s within five attempts, so the 30 s cap
  is never applied.
- The heartbeat is application-level `ping`/`pong` text frames, not RFC 6455
  control frames. The server must be written to answer them.
- The chat is single-sided: every message renders as the local user's, and
  `isSelf` on `Chat` is declared and never read. The `ForEach` key is
  `${JSON.stringify(item)}_${index}`, which changes whenever `isSuccessed`
  flips - the `@Trace` field does the real update work and the key fights it.
- `NetworkMonitor` has no teardown; the `NetConnection` outlives the page.

## Pitfalls

- **`HW-14-0044` (B/high, confirmed) — the advertised weak-network reconnect
  is dead code.** The `netAvailable` handler requires `!this.ws`, but `ws` is
  only nulled by the manual `close()` (which also blocks reconnection), so
  after a drop it still holds the stale socket. `retryCount` is never reset on
  recovery either, so once the five attempts are spent during an outage the
  app stays offline until restart. Fix: track an explicit `isConnected`, and
  reset `retryCount` when the network returns.
- **`HW-14-0045` (B/high, confirmed) — reconnect churn loop.** `ac_connect`
  closes the previous socket *after* creating the new one has begun; the old
  socket's `close` handler schedules a retry; `handleOpen` resets the counter
  but never cancels the pending timer; the timer fires and tears down the
  healthy socket. A single heartbeat timeout puts the manager into a permanent
  1-2 second connect/close cycle, and the retry cap never engages because the
  counter keeps resetting. Fix: `off` the old socket's listeners before
  closing it, and `clearTimeout(this.retryTimer)` in `handleOpen`.
- **`HW-14-0046` (B/medium, confirmed) — the offline resend cache never
  drains.** `this.loadingData.slice(index, index)` is a no-op where
  `splice(index, 1)` was intended (and `if (this.loadingData)` is always
  true), so every once-failed message is re-sent on every reconnect and the
  duplicates accumulate. Fix: copy the queue, clear it, then send.
- **`HW-14-0047` (B/low, confirmed) — heartbeat frames fall through into the
  message parser.** There is no `return` after the `ping`/`pong` branches, so
  each heartbeat emits a bogus `websocketMessage` with `msgID` `'ping'` or
  `'pong'`. The `pong` reply is also sent without a `.catch`, which is an
  unhandled rejection on a failing socket. Fix: return after control handling;
  catch the send.
- **`HW-14-0002` (D/low, confirmed, systematic) — the sample and the doc teach
  cleartext `ws://`.** Doc step 1 and `Chat.ets:42` both use
  `ws://echo.websocket.org`; `SOCIAL-44`'s `WebSocketClient2` defaults to
  `ws://192.168.137.1:8000`. Neither page mentions TLS. An IM reliability
  template is exactly where transport security should be modelled, and the
  public echo service is `wss`-only today, so the cleartext URL also simply
  fails. Fix: `wss://`, with a note on certificate requirements.
- **Not filed:** `ws.connect().catch` only logs. A connect that rejects
  without producing a `close` or `error` event leaves nothing to schedule a
  retry; call `_scheduleReconnect()` from that catch.

## References

- `huawei_industry_tree/14_social_communication/docs/20_weak_network_reconnection.md` - the source page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/weak_network_reconnection-0000002288904746
- `documentation/harmonyos-references/03_system/js-apis-websocket.md` - `createWebSocket`, `connect`, `send`, `close`, `on/off('open'|'message'|'close'|'error')`, `WebSocketRequestOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-websocket
- `documentation/harmonyos-guides/04_system/websocket-connection.md` - the client lifecycle and closing codes
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/websocket-connection
- `documentation/harmonyos-references/03_system/errorcode-net-websocket.md` - what the `BusinessError` codes on send/connect mean
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/errorcode-net-websocket
- `documentation/harmonyos-references/03_system/js-apis-net-connection.md` - `createNetConnection`, `register`, `on('netAvailable')`, `on('netUnavailable')`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-net-connection
- `documentation/harmonyos-references/03_system/js-apis-emitter.md` - `emitter.on`, `emit`, `off` and `EventData`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-emitter
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-richeditor.md` - `RichEditorController`, `getSpans`, `deleteSpans`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-richeditor
- `documentation/harmonyos-references/02_application-framework/js-apis-util.md` - `generateRandomUUID`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-util
- `SOCIAL-44` - `WebSocketClient2`, the other cleartext-`ws://` sample (`HW-14-0002`)
