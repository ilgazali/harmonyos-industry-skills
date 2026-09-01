---
id: UTIL-40
title: Native ping - OH_NetConn_QueryProbeResult on a napi async work, surfaced to ArkTS as a Promise
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/40_ping_tool.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/ping_tool-0000002506881396
sample: huawei_industry_tree/15_utilities/downloads/PingTool.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [hilog, util, window, napi_create_promise, napi_create_async_work, napi_queue_async_work, napi_delete_async_work, napi_resolve_deferred, napi_reject_deferred, napi_get_value_string_utf8, OH_NetConn_QueryProbeResult, NetConn_ProbeResultInfo, TextInput, TextArea]
permissions: [ohos.permission.INTERNET]
min_api: 20
modules: [entry (entry)]
findings: [HW-15-0083, HW-15-0101, HW-15-0102]
status: verified-with-fixes
---

## When to use

Load this card when you need to **run a blocking C API from ArkTS without
freezing the UI**. Ping is the example, but the shape is the general one: a
native function that takes seconds to return, wrapped in a Node-API async work,
exposed to ArkTS as a `Promise<string>`. The same skeleton covers a native
image transform, a checksum over a large file, a crypto operation, or any
NDK call whose reference page says "do not call it in the main process".

The specific capability is also worth knowing about. `OH_NetConn_QueryProbeResult`
from `net_connection.h` gives an app real ICMP-style reachability data - loss
rate and min/max/average/standard-deviation RTT - without raw sockets and
without a system permission beyond `ohos.permission.INTERNET`. Its own reference
page states plainly that it must not run on the main thread, which is why the
async-work wrapper is not optional here.

**`HW-15-0083` is a use-after-free and it is reproduced verbatim in the
document.** Anyone who copies the published snippet inherits it. Read the
corrected `NativePing` below before you copy anything from doc 40 step 3.

## Feature checklist

- A single page: title, a rounded address input with a search icon, a capsule
  检测 button and a read-only results area.
- The input accepts an IPv4 dotted quad or a domain name; validation is
  deferred until blur so typing is not fought.
- An invalid address turns the input border red and reveals a red hint line.
- The button re-validates before firing and refuses to call native code on a
  bad address.
- Each run appends a formatted block to the results area: loss percentage and
  the four RTT figures.
- Runs accumulate; the area starts with an empty-state illustration and a hint,
  which disappear on the first result.
- The page padding tracks the status-bar and navigation-indicator avoid areas.

## Architecture

One `entry` module with an ArkTS page over a two-file native library.

```
entry/src/main
├── cpp
│   ├── CMakeLists.txt              builds libentry.so against libnet_connection.so
│   ├── napi_init.cpp               CallbackData, the three napi callbacks, module registration
│   ├── tool/PingTool.h             class PingTool { static char *Ping(char[], int32_t); }
│   ├── tool/PingTool.cpp           OH_NetConn_QueryProbeResult -> hand-built JSON string
│   └── types/libentry/
│       ├── Index.d.ts              nativePing: (address, duration) => Promise<string>
│       └── oh-package.json5        names the package "libentry.so"
├── ets
│   ├── common/CommonConstants.ets  the printf template and '100%'
│   ├── entryability/EntryAbility.ets   avoid areas -> AppStorage, avoidAreaChange subscription
│   ├── entrybackupability/EntryBackupAbility.ets
│   ├── pages/MainPage.ets          @Entry: input, validation, the call, the results area
│   └── utils/Logger.ets            hilog wrapper
└── module.json5                    one permission: INTERNET
```

The documented 工程目录 lists only the `ets` subtree. It is not wrong about
what it shows, but it omits `cpp/` entirely - the half of the project the
article is actually about, and the half carrying the defect.

**The design decision worth copying** is the boundary. The native side does one
thing: it returns a `char*` holding a JSON object, or `nullptr`. It does not
know about promises' semantics, does not format for display, does not localise.
`napi_init.cpp` is pure plumbing over that, and `MainPage.ets` does
`JSON.parse` into a `PingResult` interface and `util.format` into a template
kept in `CommonConstants`. Three layers, each replaceable: swap the probe for a
traceroute and only `PingTool.cpp` changes; retheme the output and only the
template changes.

Hand-rolling JSON with an `ostringstream` is defensible for five numeric fields
with no user-controlled content. It would not be if the address were echoed into
the string - that would need escaping.

**The structural trap** is `CallbackData::result` being a raw `char*` owned by
whoever remembers to free it. `PingExecuteCB` assigns the pointer
`PingTool::Ping` allocated with `new char[]`; `ReleaseCallbackData` frees it;
and in between, three separate early-return paths in `NativePing` have to get
the ownership right. Two of them do not (`HW-15-0083`). A `std::unique_ptr<char[]>`
member, or a `std::string`, removes both the leak and the double-management.

## Implementation steps

1. **Declare the native signature in `types/libentry/Index.d.ts`** as
   `export const nativePing: (address: string, duration: number) => Promise<string>;`
   and point `oh-package.json5`'s `types` at it. Without it the ArkTS import has
   no type.
2. **Wire the build** in `entry/build-profile.json5` with
   `externalNativeOptions.path` pointing at `src/main/cpp/CMakeLists.txt`, and
   link `libnet_connection.so` alongside `libace_napi.z.so` and
   `libhilog_ndk.z.so`.
3. **Validate arity and types first** in `NativePing` - `argc < 2`, then
   `napi_typeof` for `napi_string` and `napi_number` - and `napi_throw_error`
   before allocating anything.
4. **Read the string in two passes**: length with a null buffer, then the data
   into `new char[strLen + 1]`. **Free that buffer on every failure path after
   it exists**, including the duration read (`HW-15-0083`).
5. **Create the promise before the async work**, so the deferred can be stored
   on the `CallbackData` the work will carry.
6. **On a queue failure, delete the async work before freeing the struct that
   holds its handle** (`HW-15-0083`). The published order does the opposite.
7. **Reject with a coded error object, not a bare string**, so the ArkTS
   `.catch((error: BusinessError) => ...)` can read `error.code` and
   `error.message` (`HW-15-0083`).
8. **Keep `OH_NetConn_QueryProbeResult` in `PingExecuteCB`**, never in the
   complete callback - the execute callback is the only one that runs off the
   ArkTS thread.

## Verified snippets

All snippets are from `PingTool.zip`. Corrected forms are marked.

**Entry point and queueing — `entry/src/main/cpp/napi_init.cpp`**
(corrected, see `HW-15-0083`)

```cpp
struct CallbackData {
    napi_async_work asyncWork = nullptr;
    napi_deferred deferred = nullptr;
    napi_ref callback = nullptr;
    char *argAddress = nullptr;
    int32_t argDuration = 3;
    char* result = nullptr;
};

// 读取探测的持续时间
napi_status durationStatus;
int32_t duration;
durationStatus = napi_get_value_int32(env, args[1], &duration);
if (durationStatus != napi_ok) {
    delete[] buffer;                                  // FIX: the sample leaks the address buffer here
    napi_throw_error(env, nullptr, "read duration data error");
    return nullptr;
}

// 创建异步任务，执行ping操作
auto callbackData = new CallbackData();
callbackData->deferred = deferred;
callbackData->argAddress = buffer;
callbackData->argDuration = duration;

napi_status createAsyncStatus;
createAsyncStatus = napi_create_async_work(env, nullptr, resourceName, PingExecuteCB, PingCompleteCB, callbackData,
                                           &callbackData->asyncWork);
if (createAsyncStatus != napi_ok) {
    ReleaseCallbackData(callbackData);
    napi_throw_error(env, nullptr, "create async work error");
    return nullptr;
}

if (napi_queue_async_work(env, callbackData->asyncWork) != napi_ok) {
    napi_delete_async_work(env, callbackData->asyncWork);   // FIX: must precede the delete
    ReleaseCallbackData(callbackData);                      // FIX: sample calls this first
    napi_throw_error(env, nullptr, "enqueue async work error");
    return nullptr;
}

return promiseValue;
```

**The two lines are swapped in the shipped code and in the document.**
`ReleaseCallbackData` ends with `delete callbackData`, so once it returns the
struct is freed memory. The very next statement,
`napi_delete_async_work(env, callbackData->asyncWork)`, dereferences that freed
struct to read the handle. It is a textbook use-after-free: usually harmless
because the allocator has not yet reused the page, and occasionally a crash or a
double free of the async work. Doc 40 lines 91-96 ship exactly this order, so
the defect propagates to every reader who copies the published snippet.

The leak on the duration path is smaller but real: `buffer` was allocated three
statements earlier and this is the one early return that does not free it. Every
malformed second argument leaks the address string.

Note the correct pattern the sample does get right two blocks up: the
`createResourceStatus` and `createAsyncStatus` failures call
`ReleaseCallbackData` **only**, because at that point no async work exists to
delete.

**Completion, resolution and rejection — same file**
(corrected, see `HW-15-0083`)

```cpp
static void PingCompleteCB(napi_env env, napi_status status, void *data)
{
    CallbackData *callbackData = reinterpret_cast<CallbackData *>(data);

    // FIX: the sample calls napi_create_string_utf8 on a possibly-null result and
    //      infers failure from the create call failing
    if (callbackData->result == nullptr) {
        napi_value code = nullptr;
        napi_value message = nullptr;
        napi_value error = nullptr;
        napi_create_string_utf8(env, "2100003", NAPI_AUTO_LENGTH, &code);
        napi_create_string_utf8(env, "ping probe failed", NAPI_AUTO_LENGTH, &message);
        napi_create_error(env, code, message, &error);
        napi_reject_deferred(env, callbackData->deferred, error);
    } else {
        napi_value result = nullptr;
        napi_status createResStatus =
            napi_create_string_utf8(env, callbackData->result, NAPI_AUTO_LENGTH, &result);
        if (createResStatus != napi_ok) {
            OH_LOG_Print(LOG_APP, LOG_ERROR, LOG_DOMAIN, LOG_TAG, "create async work result value error");
            napi_value error = nullptr;
            napi_create_error(env, nullptr, result, &error);
            napi_reject_deferred(env, callbackData->deferred, error);
        } else {
            napi_resolve_deferred(env, callbackData->deferred, result);
        }
    }

    napi_delete_async_work(env, callbackData->asyncWork);
    ReleaseCallbackData(callbackData);
}
```

**The shipped control flow detects a failed ping by accident.** `PingTool::Ping`
returns `nullptr` when `OH_NetConn_QueryProbeResult` fails, and the sample's
only reaction is that `napi_create_string_utf8(env, nullptr, ...)` also fails,
which sends it down the "create result value error" branch. Two different
conditions - a genuinely unreachable host and a napi allocation failure - share
one path, and the `if (callbackData->result == nullptr)` check inside the
success branch is dead code that can never run. Testing the pointer explicitly,
first, separates them.

**The rejection value is the user-visible half.** `napi_reject_deferred` takes
any `napi_value`, and the sample passes a plain string. `MainPage.ets` then does
`` `ping failed: ${error.code} ${error.message}` `` on it - a JS string has
neither property, so a failed ping renders literally as
`ping failed: undefined undefined`. `napi_create_error` produces an object with
both, which is what the ArkTS `BusinessError` type annotation in the page
already assumes.

Note also that `napi_delete_async_work` here comes **before**
`ReleaseCallbackData`, which is the correct order - the same order the queue
failure path gets wrong.

**The probe itself — `entry/src/main/cpp/tool/PingTool.cpp`** (as shipped)

```cpp
char* PingTool::Ping(char address[], int32_t duration)
{
    NetConn_ProbeResultInfo probeInfo;
    probeInfo.lossRate = 0;
    probeInfo.rtt[0] = 0;
    probeInfo.rtt[1] = 0;
    probeInfo.rtt[2] = 0;
    probeInfo.rtt[3] = 0;

    int32_t ret = OH_NetConn_QueryProbeResult(address, duration, &probeInfo);

    // ret!=0时代表调用出错
    if (ret != 0) {
        OH_LOG_ERROR(LOG_APP, "query probe info error: ret = %{public}d", ret);
        return nullptr;
    }
    // 成功调用后，获取返回结果中的丢包率，最小、最大、平均以及标准时延
    uint8_t lossRate = probeInfo.lossRate;
    uint32_t minDelay = probeInfo.rtt[0];
    uint32_t maxDelay = probeInfo.rtt[1];
    uint32_t avgDelay = probeInfo.rtt[2];
    uint32_t stdDelay = probeInfo.rtt[3];
    char *json = nullptr;
    try {
        std::ostringstream oss;
        oss << "{\"lossRate\": " << static_cast<int>(lossRate) << ", \"minDelay\": " << minDelay
            << ", \"maxDelay\": " << maxDelay << ", \"avgDelay\": " << avgDelay << ", \"stdDelay\": " << stdDelay
            << "}";
        std::string str = oss.str();
        json = new char[str.size() + 1];
        str.copy(json, str.size(), 0);
        json[str.size()] = '\0';
    } catch (const std::bad_alloc& e) {
        OH_LOG_Print(LOG_APP, LOG_ERROR, LOG_DOMAIN, LOG_TAG, "build json result error: %{public}s", e.what());
    }
    return json;
}
```

**Three details in this function matter.** The `static_cast<int>` on `lossRate`
is not cosmetic: `uint8_t` is a character type, so streaming it directly would
write a control byte into the JSON instead of a number. The zero-initialisation
of the whole `probeInfo` before the call means a partially-filled result reads
as zeros rather than stack garbage. And the `bad_alloc` catch leaves `json` as
`nullptr` and falls through to the same return, so an allocation failure and a
probe failure are indistinguishable to the caller - acceptable only because
both end in a rejected promise.

`duration` is a **duration in seconds, not a packet count**: the reference
states the probe interval is one second, so passing `2` means roughly two
probes. `MainPage` hardcodes that `2` at the call site.

**Validation and the call — `entry/src/main/ets/pages/MainPage.ets`** (as shipped)

```typescript
import PingTool from 'libentry.so';

isValidAddress(input: string) {
  const IP_REGEX = /^(25[0-5]|2[0-4][0-9]|[1-9]?[0-9])(\.(25[0-5]|2[0-4][0-9]|[1-9]?[0-9])){3}$/;
  const DOMAIN_REGEX = /^([a-zA-Z0-9]([a-zA-Z0-9\-]{0,61}[a-zA-Z0-9])?\.)+[a-zA-Z]{2,6}$/;
  return IP_REGEX.test(input) || DOMAIN_REGEX.test(input);
}

// ...
Button($r('app.string.ping_button_tip'), { type: ButtonType.Capsule })
  .onClick(() => {
    if (!this.isValidAddress(this.pingAddress)) {
      this.isValid = false;
      return;
    }
    try {
      // 调用Native侧的接口实现ping功能
      PingTool.nativePing(this.pingAddress, 2)
        .then((result) => {
          let parseResult = JSON.parse(result) as PingResult;
          this.pingMsg += util.format(CommonConstants.PING_MESSAGE_TEMPLATE, this.pingAddress,
            parseResult.lossRate, parseResult.minDelay,
            parseResult.maxDelay, parseResult.avgDelay, parseResult.stdDelay);
          this.pingMsg += '\n\n';
        }).catch((error: BusinessError) => {
        this.pingMsg += `ping failed: ${error.code} ${error.message}`;
        Logger.error('MainPage', `ping failed: ${error.code} ${error.message}`);
      });
    } catch (e) {
      Logger.error('MainPage', `nativePing error: ${e.message}`);
    }
  });
```

**Validation happens twice, deliberately.** `onChange` sets `isValid = true`
unconditionally so the red border never fights the user mid-typing; `onBlur`
runs the real check once focus leaves; and the button checks again before
crossing into native code, because a user can tap 检测 without ever blurring the
field. The empty string is special-cased in `onBlur` to the valid state so an
untouched form looks clean.

The `try/catch` around the call catches a **synchronous** throw from the native
binding - a wrong argument type reaches `napi_throw_type_error` and surfaces
here - while the `.catch` handles the rejected promise. Both are needed; they
cover different failure classes. Note that neither disables the button while a
probe is in flight, so a user can queue several concurrent async works.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.INTERNET" }
]
```

- `INTERNET` is `system_grant` - no `reason`, no `usedScene`, no runtime
  request. This is the whole permission surface, and it matches the document's
  权限说明 exactly.
- `OH_NetConn_QueryProbeResult` requires exactly this permission and returns
  `201` if it is missing. Sibling `OH_NetConn_QueryTraceRoute` additionally
  needs `LOCATION` and `ACCESS_NET_TRACE_INFO`, so a traceroute feature is not a
  drop-in extension of this sample.
- `deviceTypes` is `phone` only.
- `abiFilters` is `["arm64-v8a", "x86_64"]`; add `armeabi-v7a` if you target
  32-bit devices, or the library will be missing at runtime.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `OH_NetConn_QueryProbeResult` is
  **since API 20** - there is no fallback for earlier devices.
- The reference describes `NetConn_ProbeResultInfo.rtt` as "round-trip time in
  ms, including the maximum, minimum, average, and standard deviations", while
  the sample reads `rtt[0]` as the minimum and `rtt[1]` as the maximum, and the
  display template labels all four as μs. Verify the index order and the unit
  against a known host before trusting the numbers; do not carry the labels over
  unchecked.
- The domain regex caps the TLD at six characters, so `.museum` passes and
  longer new TLDs do not. It also rejects IPv6 entirely.
- `duration` is fixed at `2` in the page; there is no UI to change it.
- `pingMsg` grows without bound - every run appends and nothing trims or offers
  a clear. It is bound to an editable `TextArea`, so a user can also type into
  the results pane.
- Runs are not serialised: nothing disables the button while a probe is running.
- The native library is named `entry` in both `CMakeLists.txt` and
  `napi_module.nm_modname`; these must agree or the module will not register.

## Pitfalls

- **`HW-15-0083`** (B/high, confirmed): on the `napi_queue_async_work` failure
  path `ReleaseCallbackData(callbackData)` runs before
  `napi_delete_async_work(env, callbackData->asyncWork)`, so the handle is read
  out of freed memory - and doc 40 lines 91-96 publish the same order, spreading
  it to readers. In the same file the address buffer is leaked when the duration
  argument fails to read, and a failed ping is signalled only by
  `napi_create_string_utf8` choking on a `nullptr`, then rejected with a plain
  string so the page's `` `${error.code} ${error.message}` `` renders
  `ping failed: undefined undefined`. Fix: delete the async work before freeing
  the struct, free the buffer on that path, test `result == nullptr` explicitly
  and reject with `napi_create_error`.

## References

- `documentation/harmonyos-guides/10_ndk-development/use-napi-asynchronous-task.md` - the promise-flavoured async work skeleton and its cleanup order
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/use-napi-asynchronous-task
- `documentation/harmonyos-references/06_standard-libraries/napi.md` - which Node-API surface HarmonyOS supports, and the library to link
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/napi
- `documentation/harmonyos-references/03_system/capi-net-connection-h.md` - `OH_NetConn_QueryProbeResult`, its parameters, return codes and the "do not call on the main thread" note
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/capi-net-connection-h
- `documentation/harmonyos-references/03_system/capi-netconnection-netconn-proberesultinfo.md` - `lossRate` and the `rtt` array
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/capi-netconnection-netconn-proberesultinfo
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` - `ohos.permission.INTERNET`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
