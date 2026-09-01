---
id: COMMON-47
title: Charset conversion with ICU4C - UTF-8 to GB2312 through a NAPI bridge
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/47_format_conversion.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/format_conversion-0000002516884120
sample: huawei_industry_tree/19_common_technical_solutions/downloads/FormatConversion.zip
kits: ["@kit.ArkTS", "@kit.ArkUI", "@kit.IMEKit"]
apis: [ucnv_open, ucnv_close, ucnv_convertEx, u_errorName, U_FAILURE, U_BUFFER_OVERFLOW_ERROR, napi_get_value_string_utf8, napi_create_buffer, napi_define_properties, napi_module_register, "buffer.from", Uint8Array, TextArea, "inputMethod.getController", "InputMethodController.stopInputSession"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-19-0153, HW-19-0154, HW-19-0155, HW-19-0156, HW-19-0157, HW-19-0158, HW-19-0180, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when text must cross a boundary that does not speak UTF-8 -
a legacy server, a printer or scale, a fixed-format file, a hardware protocol
that specifies GB2312/GBK. HarmonyOS applications are UTF-8 throughout, and
ArkTS has no built-in encoder for legacy Chinese charsets, so the conversion has
to go through ICU4C in a native module.

Read the Pitfalls section before adopting this code. The native layer contains a
use-after-free, a heap overflow, and a leak, and the ArkTS layer re-encodes the
result as UTF-8 - so the conversion the sample displays is not the conversion it
computed.

## Feature checklist

- Open one `UConverter` per charset with `ucnv_open`, and close each exactly once
  (HW-19-0153).
- Size the target buffer for the worst case **plus one byte** for the terminator
  (HW-19-0154).
- Treat `U_BUFFER_OVERFLOW_ERROR` as a failure, not as success (HW-19-0155).
- Return the bytes to ArkTS as a buffer, and **declare that type** in
  `Index.d.ts` (HW-19-0156).
- Free every native allocation on every path (HW-19-0157).

## Architecture

Single-module project (`entry` HAP) with a native library:

| File | Responsibility |
| --- | --- |
| `cpp/FormatConversion.h` | the `CharsetConverter` struct and five function declarations |
| `cpp/FormatConversion.cpp` | `ConverterInit`, `ConverterCleanup`, `Utf8ToGb2312`, `Gb2312ToUtf8`, `FreeConversionBuffer` |
| `cpp/napi_init.cpp` | the NAPI wrappers, the demo, and module registration |
| `cpp/types/libentry/Index.d.ts` | the ArkTS-visible signatures |
| `cpp/CMakeLists.txt` | builds `libentry.so` |
| `pages/Index.ets` | input `TextArea`, converted text, hex dump |

The document's tree lists only the first two (HW-19-0158).

**Why `ucnv_convertEx` and not `ucnv_fromUChars` + `ucnv_toUChars`.**
`convertEx` performs the whole source-charset → UTF-16 pivot → target-charset
chain in one call, using an internal pivot buffer when the four pivot pointers
are `NULL`. That is exactly the `UTF-8 → Unicode → GB2312` path the document
describes, without the caller having to allocate or size a UTF-16 intermediate.

**The two `TRUE` arguments matter.** They are `reset` and `flush`. Passing both
means each call is a complete, self-contained conversion - correct for a
one-shot API, and the reason the converters can be reused across calls without
carrying state.

**Buffer sizing is per-direction and asymmetric.**

| direction | worst case | what the code allocates |
| --- | --- | --- |
| UTF-8 → GB2312 | 2 bytes out per character | `utf8Len * 2`, malloc `+ 1` — correct |
| GB2312 → UTF-8 | 3 bytes out per 2 bytes in | `gb2312Len * 3 + 1`, malloc that — **off by one** (HW-19-0154) |

The first function reserves the terminator byte beyond the conversion limit; the
second folds the `+ 1` into the limit itself, so the terminator lands outside the
allocation. Comparing the two side by side is the clearest way to see the bug.

**The `gb2312` → `gbk` fallback is the riskiest code in the file.** GB2312 is a
subset of GBK, so falling back is sensible - but the fallback path closes the
UTF-8 converter, which has nothing to do with the failure (HW-19-0153).

**The NAPI boundary returns bytes, not text.** `napi_create_buffer` is the right
choice: GB2312 bytes are not valid UTF-8, so they cannot survive a string
round-trip. The `Index.d.ts` declaration says `string` anyway, and the page acts
on that (HW-19-0156).

**One nice touch on the ArkTS side.** `.onTouch` dismisses the soft keyboard
through `inputMethod.getController().stopInputSession()`, wrapped in both a
`try/catch` and a `.catch` - the synchronous throw and the promise rejection are
different failures and both are handled.

## Implementation steps

1. **Declare the converter pair** as a struct holding one `UConverter*` per
   charset.
2. **Open both** with `ucnv_open`, checking `U_FAILURE` after each. On a
   fallback, do not release the converter that opened successfully, and null out
   pointers you do close (HW-19-0153).
3. **Size the target**: worst-case expansion for the direction, allocated one
   byte larger than the limit you pass to ICU (HW-19-0154).
4. **Convert** with `ucnv_convertEx(targetConv, sourceConv, &targetPtr,
   targetLimit, &sourcePtr, sourceLimit, NULL, NULL, NULL, NULL, TRUE, TRUE,
   &status)`.
5. **Check the status with `U_FAILURE` alone** - a buffer overflow is a failure
   (HW-19-0155).
6. **Terminate, report the length, hand back the pointer**, and give the caller a
   matching free function.
7. **Bridge to ArkTS**: query the input string's length before allocating,
   convert, copy into a `napi_create_buffer`, free everything (HW-19-0157).
8. **Declare the real return type** in `Index.d.ts` and consume the bytes without
   re-encoding (HW-19-0156).

## Verified snippets

All snippets below come from the sample project, not from the document.

### The converter pair

`FormatConversion.zip#FormatConversion/entry/src/main/cpp/FormatConversion.h`

```cpp
#include <cstddef>
#include "unicode/ucnv.h"

typedef struct {
    UConverter* utf8Conv;
    UConverter* gb2312Conv;
} CharsetConverter;

int ConverterInit(CharsetConverter* converter);
void ConverterCleanup(CharsetConverter* converter);
int Utf8ToGb2312(CharsetConverter* converter, const char* utf8Str, int utf8Len, char** gb2312Str, size_t* gb2312Len);
int Gb2312ToUtf8(CharsetConverter* converter, const char* gb2312Str, int gb2312Len, char** utf8Str, size_t* utf8Len);
void FreeConversionBuffer(char* buffer);
```

`FreeConversionBuffer` existing at all is good practice: the buffers are
allocated by the library and must be released by it, not by whatever allocator
the caller happens to use.

### Opening the converters

`FormatConversion.zip#FormatConversion/entry/src/main/cpp/FormatConversion.cpp`

```cpp
int ConverterInit(CharsetConverter* converter)
{
    UErrorCode status = U_ZERO_ERROR;

    converter->utf8Conv = ucnv_open("utf-8", &status);
    if (U_FAILURE(status)) {
        OH_LOG_ERROR(LOG_APP, "Failed to open UTF-8 converter: %{public}s.", u_errorName(status));
        return -1;
    }

    converter->gb2312Conv = ucnv_open("gb2312", &status);
    if (U_FAILURE(status)) {
        // 如果gb2312不可用，尝试gbk
        ucnv_close(converter->utf8Conv);          // FIX (HW-19-0153): remove this line
        converter->gb2312Conv = ucnv_open("gbk", &status);
        if (U_FAILURE(status)) {
            OH_LOG_ERROR(LOG_APP, "Failed to open GB2312/GBK converter: %{public}s.", u_errorName(status));
            ucnv_close(converter->utf8Conv);      // and this one then closes it a second time
            return -1;
        }
    }

    return 0;
}

void ConverterCleanup(CharsetConverter* converter)
{
    if (converter->utf8Conv) {
        ucnv_close(converter->utf8Conv);
    }
    if (converter->gb2312Conv) {
        ucnv_close(converter->gb2312Conv);
    }
}
```

`ConverterCleanup` tests the pointers, which only helps if closed pointers are
nulled - they are not.

### UTF-8 to GB2312 (the correctly sized one)

`FormatConversion.zip#FormatConversion/entry/src/main/cpp/FormatConversion.cpp`

```cpp
int Utf8ToGb2312(CharsetConverter* converter, const char* utf8Str, int utf8Len, char** gb2312Str, size_t* gb2312Len)
{
    UErrorCode status = U_ZERO_ERROR;

    if (!utf8Str || utf8Len <= 0) {
        return -1;
    }

    int32_t targetCapacity = utf8Len * 2;
    char* target = (char*)malloc(targetCapacity + 1);      // note: capacity + 1
    if (!target) {
        return -1;
    }

    const char* sourcePptr = utf8Str;
    const char* sourceLimit = utf8Str + utf8Len;
    char* targetPtr = target;
    char* targetLimit = target + targetCapacity;           // limit stops one short

    // 执行转换：UTF-8 → Unicode → GB2312
    status = U_ZERO_ERROR;
    ucnv_convertEx(converter->gb2312Conv, converter->utf8Conv, &targetPtr, targetLimit, &sourcePptr, sourceLimit, NULL,
        NULL, NULL, NULL, TRUE, TRUE, &status);

    if (U_FAILURE(status) && status != U_BUFFER_OVERFLOW_ERROR) {   // FIX (HW-19-0155)
        free(target);
        OH_LOG_ERROR(LOG_APP, "UTF-8 to GB2312 conversion failed: %{public}s.", u_errorName(status));
        return -1;
    }

    *targetPtr = 0;
    *gb2312Len = targetPtr - target;
    *gb2312Str = target;

    return 0;
}
```

Argument order in `ucnv_convertEx` is **target converter first, source converter
second** - `gb2312Conv` then `utf8Conv` here, and reversed in the other
direction. It is easy to transpose and produces silently wrong output when you
do.

### GB2312 to UTF-8 (the off-by-one)

`FormatConversion.zip#FormatConversion/entry/src/main/cpp/FormatConversion.cpp`

```cpp
int32_t targetCapacity = gb2312Len * 3 + 1;      // FIX (HW-19-0154): the +1 belongs
char* target = (char*)malloc(targetCapacity);    // on the malloc, not the capacity
...
char* targetLimit = target + targetCapacity;     // == end of the allocation
...
*targetPtr = 0;                                  // may write target[targetCapacity]
```

Corrected, matching its sibling:

```cpp
int32_t targetCapacity = gb2312Len * 3;
char* target = (char*)malloc(targetCapacity + 1);
...
char* targetLimit = target + targetCapacity;
```

### The NAPI wrapper

`FormatConversion.zip#FormatConversion/entry/src/main/cpp/napi_init.cpp`

```cpp
static napi_value Utf82Gb2312(napi_env env, napi_callback_info info)
{
    size_t argc = 1;
    napi_value args[1] = {nullptr};

    napi_get_cb_info(env, info, &argc, args, nullptr, nullptr);

    napi_valuetype valuetype0;
    napi_typeof(env, args[0], &valuetype0);        // result never checked

    const int bufferLen = 1024;
    char *utf8Text = (char*)malloc(sizeof(char) * bufferLen);   // FIX (HW-19-0157): never freed
    size_t len;
    napi_get_value_string_utf8(env, args[0], utf8Text, bufferLen, &len);

    CharsetConverter converter;
    if (ConverterInit(&converter) != 0) {
        OH_LOG_ERROR(LOG_APP, "Failed to initialize converters.");
        return 0;
    }

    char* gb2312Text = nullptr;
    size_t gb2312Len = 0;

    if (Utf8ToGb2312(&converter, utf8Text, strlen(utf8Text), &gb2312Text, &gb2312Len) == 0) {
        for (int i = 0; i < gb2312Len; i++) {
            OH_LOG_INFO(LOG_APP, "%{public}x.", gb2312Text[i]);
        }
    }

    void *bufferPtr = nullptr;
    size_t bufferSize = gb2312Len;
    napi_value buffer = nullptr;
    // 调用napi_create_buffer接口创建并获取一个指定大小的ArkTS Buffer
    napi_status status = napi_create_buffer(env, bufferSize + 1, &bufferPtr, &buffer);
    if (status != napi_ok) {
        OH_LOG_ERROR(LOG_APP, "napi_create_buffer failed");
        return nullptr;
    }
    memcpy((char *)bufferPtr, gb2312Text, gb2312Len);   // last byte left unwritten
    FreeConversionBuffer(gb2312Text);
    ConverterCleanup(&converter);

    return buffer;
}
```

Note `strlen(utf8Text)` is used as the length even though
`napi_get_value_string_utf8` already reported it in `len` - the reported value is
computed and then ignored.

### Module registration

`FormatConversion.zip#FormatConversion/entry/src/main/cpp/napi_init.cpp`

```cpp
EXTERN_C_START
static napi_value Init(napi_env env, napi_value exports)
{
    napi_property_descriptor desc[] = {
        { "formatConversion", nullptr, FormatConversion, nullptr, nullptr, nullptr, napi_default, nullptr },
        { "utf82gb2312", nullptr, Utf82Gb2312, nullptr, nullptr, nullptr, napi_default, nullptr }
    };
    napi_define_properties(env, exports, sizeof(desc) / sizeof(desc[0]), desc);
    return exports;
}
EXTERN_C_END

static napi_module demoModule = {
    .nm_version = 1,
    .nm_flags = 0,
    .nm_filename = nullptr,
    .nm_register_func = Init,
    .nm_modname = "entry",
    .nm_priv = ((void*)0),
    .reserved = { 0 },
};

extern "C" __attribute__((constructor)) void RegisterEntryModule(void)
{
    napi_module_register(&demoModule);
}
```

This is the standard boilerplate and it is correct - worth keeping as the
reference shape for any NAPI module.

### The ArkTS side

`FormatConversion.zip#FormatConversion/entry/src/main/cpp/types/libentry/Index.d.ts`

```ts
export const formatConversion: () => void;
export const utf82gb2312: (str: string) => string;   // FIX (HW-19-0156): returns a buffer
```

`FormatConversion.zip#FormatConversion/entry/src/main/ets/pages/Index.ets`

```ts
import testNapi from 'libentry.so';
import { buffer } from '@kit.ArkTS';
import { inputMethod } from '@kit.IMEKit';

Button('UTF8转GB2312')
  .onClick(() => {
    if (this.inputMessage) {
      let res: string = testNapi.utf82gb2312(this.inputMessage);
      this.gb2312Str = res.toString();
      let array = this.stringToUint8Array(res);      // FIX (HW-19-0156): re-encodes UTF-8
      let hexstr = this.uint8ArrayToHexStr(array);
      this.hexStr = hexstr;
    }
  })

stringToUint8Array(str: string) {
  return new Uint8Array(buffer.from(str, 'utf-8').buffer);
}

uint8ArrayToHexStr(data: Uint8Array): string {
  let hexString = '';
  let i: number;
  for (i = 0; i < data.length; i++) {
    let char = ('00' + data[i].toString(16)).slice(-2);
    hexString += char;
  }
  return hexString;
}
```

`uint8ArrayToHexStr` itself is correct - the `('00' + …).slice(-2)` idiom
zero-pads each byte properly. It is the array fed to it that is wrong.

### Dismissing the keyboard

`FormatConversion.zip#FormatConversion/entry/src/main/ets/pages/Index.ets`

```ts
.onTouch(() => {
  try {
    let inputMethodController = inputMethod.getController();
    inputMethodController.stopInputSession().catch(() => {
    });
  } catch (error) {
  }
})
```

Both failure modes are covered, though both handlers are empty - at minimum the
`catch` blocks should log.

## Permissions & config

**No permissions are required** and none are declared. ICU4C and NAPI need none.

`build-profile.json5` pins `6.0.0(20)`. The native library is built by
`entry/src/main/cpp/CMakeLists.txt` into `libentry.so` and linked against the
system ICU4C; the module name in `napi_module.nm_modname` (`entry`) must match
the library name the ArkTS side imports (`libentry.so`).

`entry/src/mock/Libentry.mock.ets` is present so the page can be previewed
without the native library.

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **`ucnv_convertEx` takes the target converter first**, then the source.
- **All four pivot pointers must be `NULL` together** for ICU to use its internal
  pivot buffer.
- **Every `ucnv_open` needs exactly one `ucnv_close`.**
- **Buffers returned across the NAPI boundary must be copied**, not aliased - the
  native allocation is freed immediately after the `memcpy` here, which is
  correct.
- **GB2312 cannot represent every Unicode character.** Characters outside it
  become substitution bytes; the sample never surfaces that to the user.
- **GB2312 bytes are not valid UTF-8** and cannot survive being carried as a
  string.
- **Devices.** Per `module.json5`.

## Pitfalls

- **`ConverterInit` closes the UTF-8 converter when GB2312 fails to open, which is
  incorrect** - if the GBK fallback succeeds it returns a closed converter that
  every later conversion uses, and if the fallback fails it closes the same
  pointer twice. (HW-19-0153)
- **`Gb2312ToUtf8` sets its conversion limit to the end of the allocation and then
  writes the terminator past it, which is incorrect** - compare `Utf8ToGb2312` in
  the same file, which reserves the byte correctly. (HW-19-0154)
- **`U_BUFFER_OVERFLOW_ERROR` is excluded from the failure test in both functions,
  which is incorrect** - truncated output is returned with a success code and no
  log line. (HW-19-0155)
- **`Index.d.ts` declares `utf82gb2312` as returning a `string` while the
  implementation returns a buffer, which is incorrect** - and the page then builds
  its hex display by re-encoding that value as UTF-8, so what it shows is not the
  GB2312 bytes. (HW-19-0156)
- **The 1024-byte input buffer is never freed, longer input is silently truncated,
  and the returned buffer's last byte is never written - all incorrect.**
  (HW-19-0157)
- **The 工程目录 lists two of the five files under `cpp`, omitting the NAPI
  bridge, the CMake file and the type declarations, which is incomplete.**
  (HW-19-0158)
- **Do not assume `utf8Len * 2` bounds the GB2312 output.** It holds for text that
  maps cleanly; characters outside GB2312 produce substitution sequences that can
  exceed it, and the overflow is then swallowed by HW-19-0155.
- **Do not carry legacy-charset bytes through an ArkTS `string`.** Use a buffer
  or `Uint8Array` end to end.

## References

- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/icu4c -
  the ICU4C symbols HarmonyOS exposes, including `ucnv_open`, `ucnv_close`,
  `ucnv_convertEx` and `u_errorName`. (The local mirror at
  `documentation/harmonyos-references/09_standard-libraries/icu4c.md` is a stub
  and carries no API detail.)
- https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/napi-data-types-interfaces -
  `napi_get_value_string_utf8`, `napi_create_buffer`, `napi_define_properties`.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-inputmethod -
  `inputMethod.getController` and `stopInputSession`.
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/format_conversion-0000002516884120
