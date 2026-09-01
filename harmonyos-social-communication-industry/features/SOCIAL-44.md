---
id: SOCIAL-44
title: WebSocket chat client - a singleton connection with heartbeat and auto-reconnect, against a bundled NodeJS server
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/44_web_socket_client2.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/web_socket_client2-0000002550213863
sample: huawei_industry_tree/14_social_communication/downloads/WebSocketClient2.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [base, hilog, webSocket, window, "webSocket.createWebSocket", connect, send, close, "on('open')", "on('message')", "on('close')", "on('error')", CloseResult, setTimeout, setInterval]
permissions: [ohos.permission.INTERNET]
min_api: 20
modules: [entry (entry)]
findings: [HW-14-0083, HW-14-0084, HW-14-0002, HW-14-0001, HW-14-0087, HW-14-0088]
status: verified-with-fixes
---

## When to use

**Load this card when you need a long-lived socket rather than request/response**
- instant messaging, presence, live scores, a device telemetry feed. It is the
only sample in this pack that ships both ends: an ArkTS client and a NodeJS
server you run on a PC on the same LAN, so it is the right starting point when
you want to see the whole handshake rather than talk to somebody else's echo
service.

What the card is really about is the **connection lifecycle**, not the API. The
`webSocket` module is four listeners and three methods; everything hard is in
deciding what `close`, `error` and a failed `connect` each mean, which of them
should schedule a retry, and how a retry avoids stacking sockets. This sample
lays out a defensible structure - a singleton owning one socket and two timers,
`closeConnect()` as the single teardown - and then gets the error path wrong in
a way that disables the reconnect it advertises (`HW-14-0083`).

Treat the structure as the template and the timer logic as the exercise. Also
note before adopting: the endpoint is cleartext `ws://` (`HW-14-0002`), and the
client's heartbeat does not speak the server's heartbeat protocol
(`HW-14-0084`).

## Feature checklist

- The page connects on `aboutToAppear` and closes the socket on
  `aboutToDisappear`.
- One socket at a time: `initConnect` tears down any existing connection first.
- A text field plus a 发送 button send a JSON message
  (`{ type: 'normal', content: ... }`); the server acknowledges it.
- The server also supports `broadcast` (fan-out to every other client) and
  `send_to` (a private message to one `targetClientId`).
- Sent messages appear at the top of an on-screen log.
- A 20-second heartbeat keeps the connection alive while it is open.
- A dropped connection or an unreachable server schedules a reconnect three
  seconds later, and keeps retrying.
- A normal close (code 1000) does not trigger a reconnect.

## Architecture

One `entry` module, one page, one utility class - plus a `server/` folder that
is not part of the HAP.

```
entry/src/main/ets
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
├── pages/WebSocketClient.ets    @Entry: input, buttons, the log TextArea
└── util/WsClient.ets            the singleton: socket, listeners, both timers
server/index.js                  NodeJS `ws` server: clientId registry, broadcast, heartbeat
```

**The documented tree does not match the zip** (`HW-14-0001`). The 工程目录
block lists `pages/Index.ets`, but the only page in the archive is
`pages/WebSocketClient.ets`; it also spells the folder `utils` where the zip has
`util`, and `entrybackupablility`. This is one of four social project trees
that name files their zips do not contain.

**The design decision worth copying** is that the socket is owned by a
module-level singleton and the page owns none of it:

```typescript
export const wsClient = WsClient.getInstance();
```

The page calls `initConnect()`, `sendMessage()` and `closeConnect()` and holds
no socket reference. That is the right boundary for a connection whose lifetime
is longer than any one page - a second chat page can be pushed without
renegotiating, and the reconnect timer survives navigation. It also means the
singleton must be defensive about being re-entered, which is why `initConnect`
opens with `this.closeConnect()`.

**The decision worth avoiding** is the corollary: because `closeConnect()` is
both "the user is leaving" and "tear down before rebuilding", it clears the
reconnect timer as its first act. Calling it from the `error` listener - which
the sample does - cancels a retry that was scheduled microseconds earlier. One
method serving two intentions is what makes `HW-14-0083` possible.

## Implementation steps

1. **Declare `ohos.permission.INTERNET`.** It is a `system_grant` permission,
   so no request dialog, but without it `connect` fails.
2. **Wrap the socket in a singleton** that owns the instance, the reconnect
   timer and the heartbeat timer.
3. **Register `open`, `message`, `close` and `error` before calling `connect`,**
   so the connect callback cannot fire before the listeners exist.
4. **Retry only on abnormal closes:** `value.code !== 1000` in the `close`
   listener; a deliberate `close()` must not restart the loop.
5. **Do not tear the reconnect timer down from the error path**
   (`HW-14-0083`) - separate "user is leaving" from "rebuild the socket".
6. **Start the heartbeat only in the success branch of `connect`,** and clear it
   in `close`, so a dead socket does not keep firing sends.
7. **Answer the server's `heartbeat_request` with `heartbeat_response`**
   (`HW-14-0084`) rather than emitting an unrelated beat of your own.
8. **Log a message as sent only when `send` reports success** (`HW-14-0084`) -
   `send` on a null socket is a silent no-op through optional chaining.
9. **Point `wsUrl` at a `wss://` endpoint in anything shipped** (`HW-14-0002`);
   the sample's `ws://192.168.137.1:8000` is a LAN convenience.

## Verified snippets

All snippets are from `WebSocketClient2.zip`. Corrected forms are marked.

**The listener set — `entry/src/main/ets/util/WsClient.ets`** (corrected, see `HW-14-0083`)

```typescript
import webSocket from '@ohos.net.webSocket';

public async initConnect(): Promise<void> {
  this.closeConnect();                       // 关闭已有连接（避免重复创建）

  this.ws = webSocket.createWebSocket();
  if (!this.ws) {
    this.startReconnect();
    return;
  }

  this.ws.on('open', (err: BusinessError, value: Object) => { /* log status */ });

  this.ws.on('message', (err: BusinessError<void>, value: string | ArrayBuffer) => {
    this.handleServerMessage(value);
  });

  this.ws.on('close', (err: BusinessError, value: webSocket.CloseResult) => {
    if (value.code !== 1000) {               // 除正常关闭外，触发重连
      this.startReconnect();
    }
    this.clearHeartbeatTimer();
  });

  this.ws.on('error', (err: BusinessError) => {
    hilog.error(0x0000, 'testTag', `WebSocket error: ${err.message}`, err.code);
    this.clearHeartbeatTimer();              // FIX: shipped code calls closeConnect()
    this.ws = null;                          //      which also kills the pending retry
    this.startReconnect();
  });

  this.ws.connect(this.wsUrl, (err: BusinessError, value: boolean) => {
    if (!err) {
      this.startHeartbeat();
    } else {
      this.startReconnect();
      return;
    }
  });
}
```

**Three of the four listeners are about failure, and they overlap.** A server
that is not running produces an `error` *and* a failed `connect` callback; a
mid-session drop produces `error` then `close`. The shipped code routes `error`
into `closeConnect()`, whose first statement is `clearReconnectTimer()` - so the
`startReconnect()` the connect-failure branch just scheduled is cancelled before
it can fire. Worse, `closeConnect()` also calls `close()` on the socket, so the
`close` event that follows carries a normal code and the `value.code !== 1000`
guard declines to retry either. The advertised 断线重连 (disconnect and
reconnect) never runs. The corrected form releases the heartbeat, drops the
socket reference, and schedules the retry instead of cancelling it.

Registering the listeners before `connect` is not stylistic: `connect` can
complete fast enough on a LAN that a listener attached afterwards misses the
first `open`.

**Teardown and retry — same file** (as shipped)

```typescript
private readonly reconnectInterval: number = 3000;   // 断线重连间隔（3秒）
private reconnectTimer: number | null = null;

closeConnect() {
  this.clearReconnectTimer();
  this.clearHeartbeatTimer();
  this.ws?.close((err: BusinessError) => { /* log */ });
  this.ws = null;
}

private startReconnect(): void {
  this.clearReconnectTimer();
  this.reconnectTimer = setTimeout(() => {
    this.initConnect();
  }, this.reconnectInterval);
}
```

**`startReconnect` clearing its own timer first is the idempotence that makes
the design work.** Several paths can call it for one failure - the connect
callback, `close`, and (once corrected) `error` - and without that clear each
would stack another pending `initConnect`, producing a burst of sockets three
seconds later. As written, the last caller wins and exactly one retry is armed.

The retry is a fixed 3-second interval with no cap and no backoff, so an app
left on a dead network reconnects forever at 20 attempts a minute. Add
exponential backoff and a ceiling before shipping this.

`closeConnect` is correct *as a public teardown* - clearing both timers is
exactly what leaving the page should do. The bug is not in this method; it is in
calling it from a failure handler.

**The heartbeat that talks past the server — same file and `server/index.js`** (corrected, see `HW-14-0084`)

```typescript
// client, as shipped: an unsolicited beat every 20 s
private startHeartbeat(): void {
  this.clearHeartbeatTimer();
  this.heartbeatTimer = setInterval(() => {
    this.sendMessage({ 'type': 'heartbeat', 'timestamp': new Date().toISOString() });
  }, this.heartbeatInterval);
}

// corrected: answer the server's request instead
private handleServerMessage(message: string | ArrayBuffer): void {
  if (!message) {
    return;
  }
  if (typeof message === 'string' && message.includes('heartbeat_request')) {
    this.sendMessage({ 'type': 'heartbeat_response' });   // FIX: the type the server expects
  }
}
```

```javascript
// server/index.js — the protocol the client should be speaking
client.send(JSON.stringify({ type: 'heartbeat_request' }));
// ...
switch (message.type) {
  case 'heartbeat_response':
    console.log(`收到客户端 ${clientId} 心跳响应`);
    break;
```

**The keepalive works, but not the way it was designed to.** The server sends
`heartbeat_request` every 30 seconds and only recognises `heartbeat_response`;
the client never answers, and instead emits its own `heartbeat` every 20
seconds. That type falls into the server's `default` case, which replies
`message_received` - so the server's `lastHeartbeat` is refreshed as a side
effect of an unrecognised message, and each beat costs a spurious round trip.
It holds together only because the client's 20-second beat is shorter than the
server's 30-second timeout. Make the two halves agree and the design does what
it says.

**Logging a send that may not have happened — `entry/src/main/ets/pages/WebSocketClient.ets`** (corrected, see `HW-14-0084`)

```typescript
// WsClient.sendMessage, as shipped: silently a no-op when the socket is gone
public sendMessage(data: string | Record<string, string>): void {
  const sendData = typeof data === 'object' ? JSON.stringify(data) : data;
  this.ws?.send(sendData, (err: BusinessError) => { /* logs failure to hilog only */ });
}

// page, corrected: report the outcome to the caller
private sendNormalMessage() {
  if (!this.message.trim()) {
    return;
  }
  const text = this.message.trim();
  wsClient.sendMessage({ 'type': 'normal', 'content': text }, (ok: boolean) => {
    this.log = `${ok ? '' : '[未发送] '}${text}\n` + this.log;   // FIX: was logged unconditionally
  });
  this.message = '';
}
```

**`this.ws?.send(...)` is the quietest failure in the sample.** After a drop -
or before the first connection completes - `ws` is `null`, the optional chain
evaluates to `undefined`, and no callback ever runs. The page meanwhile has
already prepended the text to `this.log`, so the user reads their message on
screen as though it had gone out. A connection UI has to distinguish
"delivered to the socket" from "typed"; the minimal fix is to pass the send
result back and mark the line, and the honest one is a per-message state
(pending / sent / failed) plus an outbox that the reconnect drains.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.INTERNET"
  }
]
```

`ohos.permission.INTERNET` is `system_grant`: declaring it is enough, there is
no dialog and no `requestPermissionsFromUser` call. It is the only permission
the feature needs - and, unlike several other samples in this pack, here it is
genuinely used.

Running the server, from the document's 说明: put the phone and the PC on one
LAN (tethering to the PC's hotspot works), install NodeJS, run `npm init -y`
then `node index.js` in the `server` directory, and set `wsUrl` to the PC's LAN
address. The server binds explicitly to `192.168.137.1:8000`, so the address in
`WsClient.ets` and the `LOCAL_IP` constant in `index.js` must be changed
together.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `deviceTypes` is `phone`.
- The endpoint is cleartext `ws://` (`HW-14-0002`). Fine on a private LAN for a
  demo, unacceptable for chat traffic - and note that a real deployment behind
  TLS also needs the certificate story the document does not tell.
- Received messages are written to `hilog`, never to the UI. The visible
  `TextArea` shows only what this client typed, so the broadcast and private
  message paths cannot be observed from the phone at all.
- `sendBroadcastMessage` and `sendPrivateMessage` exist on the page but no
  control invokes them - the only button calls `sendNormalMessage`. The
  server's `broadcast` and `send_to` handlers are unreachable from the shipped
  UI.
- `handleServerMessage` wraps a `hilog` call in a try/catch and never parses
  the JSON it describes itself as parsing; there is no message model.
- The reconnect has no backoff, no attempt cap and no UI state, so the user
  gets no indication that the app is offline.
- Layout is fixed-width (328 vp with a 27 vp left margin) and will not adapt to
  a tablet or a resized 2in1 window.

## Pitfalls

- **`HW-14-0083`** (B/medium, confirmed): the `error` listener calls
  `closeConnect()`, whose first act is `clearReconnectTimer()` - cancelling the
  retry the connect-failure branch had just scheduled - and which then `close()`s
  the socket so the following `close` event carries code 1000 and is treated as
  normal. On an unreachable server or a mid-session drop the advertised
  auto-reconnect never fires. Fix: in the error path clear only the heartbeat,
  null the socket, and schedule (do not cancel) the reconnect.
- **`HW-14-0084`** (B/low, confirmed): the client sends `type: 'heartbeat'`
  while the bundled server sends `heartbeat_request` and recognises only
  `heartbeat_response`, so the designed request/response keepalive never runs
  and every beat lands in the server's default case, drawing a spurious reply;
  separately the page appends to the visible log unconditionally while
  `sendMessage` no-ops on a null socket, so after a drop users see messages
  marked as sent that were never transmitted. Fix: answer `heartbeat_request`
  with `heartbeat_response`, and gate the log on send success.
- **`HW-14-0002`** (D/low, confirmed): the sample defaults to
  `ws://192.168.137.1:8000` and neither the document nor the code mentions TLS.
  An instant-messaging template is exactly where transport security matters;
  the same finding covers `20_weak_network_reconnection`, whose `ws://echo.websocket.org`
  no longer works at all because that service is `wss`-only today. Fix: use
  `wss://` and document the certificate requirements.
- **`HW-14-0001`** (E/low, confirmed): the 工程目录 block lists
  `pages/Index.ets`, which the zip does not contain - the only page is
  `pages/WebSocketClient.ets` - and misspells `util` as `utils` and
  `entrybackupability` as `entrybackupablility`. One of four social trees that
  drifted from their zips. Fix: regenerate the tree.

## References

- `documentation/harmonyos-guides/04_system/websocket-connection.md` - `createWebSocket`, `connect`, `send`, `close`, the four events and `CloseResult`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/websocket-connection
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` - `ohos.permission.INTERNET`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
- `SOCIAL-20` - weak-network reconnection, the other socket sample and the origin of `HW-14-0002`
- `SOCIAL-13` - unread-message reminders, the natural consumer of an inbound message stream
