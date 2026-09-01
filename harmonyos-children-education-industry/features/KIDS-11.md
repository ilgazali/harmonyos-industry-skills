---
id: KIDS-11
title: Data usage monitor - interface byte counters against a stored baseline
industry: 08_children_education
doc: huawei_industry_tree/08_children_education/docs/11_data_monitor.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/data_monitor-0000002341262704
sample: huawei_industry_tree/08_children_education/downloads/DataMonitor.zip
kits: ["@kit.NetworkKit", "@kit.ArkData", "@kit.ArkUI", "@kit.PerformanceAnalysisKit"]
apis: ["statistics.getIfaceRxBytes", "statistics.getIfaceTxBytes", "preferences.getPreferencesSync", putSync, flushSync, getSync, deleteSync, Refresh, onRefreshing, Progress, ProgressType, bindSheet, TextInput, "Intl.DateTimeFormat", "@State"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-08-0083, HW-08-0084, HW-08-0085, HW-08-0086, HW-08-0087, HW-08-0088, HW-08-0089, HW-08-0090, HW-08-0120]
status: verified-with-fixes
---

## When to use

Load this card when the app must report **how much of something has been used
since a reference point**, where the platform only offers a running total:
network bytes here, and equally step counts, storage, battery-since-charge, or
any monotonic system counter turned into a quota.

The one idea worth taking is the **baseline pattern**:

```
first launch  ──> read the counter, store it              (the baseline)
every refresh ──> read the counter, subtract the baseline (usage since then)
                  quota - usage                           (what is left)
```

The platform gives a total that starts wherever it starts; the app wants "since
I was installed". Persisting one reading converts one into the other, and it
costs a single preference key.

**Eight findings.** The pattern is right and its handling of the edges is not:
the gauge inverts when the quota runs out, and nothing copes with the counter
falling below the stored baseline.

## Feature checklist

- A ring gauge showing remaining data against a configurable allowance.
- The figure in the middle, with the time it was last refreshed.
- Pull-to-refresh takes a new reading.
- A sheet to enter the monthly allowance in GB.
- The ring turns red once the allowance is exhausted.

## Architecture

One `entry` module, one page, two utility files. 458 lines.

```
entry/src/main/ets
├── constants/Constants.ets      GB_TO_MB = 1024, BYTE_TO_MB = 1048576
├── entryability/EntryAbility.ets
├── entrybackupability/
├── pages/Index.ets              the gauge, the sheet, both readings
└── utils
    ├── LoggerUtil.ets
    └── PreferencesClass.ets     getStore / save / delete / get
```

The documented tree matches the zip.

**Two readings with different lifetimes.** `firstDataMonitor` runs once ever
and persists; `tempDataMonitor` runs on every refresh and persists nothing:

```typescript
async firstDataMonitor() {
  if (this.firstData === 0) {                       // only when unset
    let rx = await statistics.getIfaceRxBytes('wlan0');
    let tx = await statistics.getIfaceTxBytes('wlan0');
    this.firstData = rx + tx;
    PreferencesClass.savePreferenceInfo(this.context, this.flag,
      PreferencesClass.DEFAULT_STORE, this.firstData);
  }
}

async tempDataMonitor() {
  let rx = await statistics.getIfaceRxBytes('wlan0');
  let tx = await statistics.getIfaceTxBytes('wlan0');
  let restData = rx + tx - this.firstData;          // used since the baseline
  this.tempData = this.dataNumber * Constants.GB_TO_MB - restData / Constants.BYTE_TO_MB;
  this.time = this.getTime();
}
```

**The unit chain is the part to get right and this sample does.** `rx`/`tx` are
bytes, the allowance is GB, and the display is MB - so the allowance is
multiplied by `GB_TO_MB` (1024) and the usage divided by `BYTE_TO_MB`
(1048576), leaving both sides in MB before they are subtracted. Two named
constants instead of `1024 * 1024` inline.

**Downlink and uplink are summed.** `getIfaceRxBytes` and `getIfaceTxBytes` are
separate calls and a data allowance counts both directions, so neither can be
dropped.

**The ordering in `aboutToAppear` matters:**

```typescript
this.firstData = PreferencesClass.getPreferenceInfo(...);   // sync: load baseline
this.firstDataMonitor().then(() => {                        // async: set it if unset
  this.tempDataMonitor();                                   // only then measure
});
```

Chaining the second read behind the first is deliberate - measuring before the
baseline exists would subtract zero and report the interface's whole lifetime
as this app's usage.

## Implementation steps

1. **Load the baseline synchronously** from preferences.
2. **If it is unset, take a reading and persist it** - and re-baseline when the
   counter goes backwards (`HW-08-0085`).
3. **Choose the interface deliberately** rather than hardcoding one
   (`HW-08-0084`).
4. **Convert bytes and GB into one unit** before subtracting.
5. **Clamp the gauge** so an exhausted allowance reads empty (`HW-08-0083`).
6. **Await the refresh** before clearing the indicator (`HW-08-0086`).
7. **Validate the allowance input** (`HW-08-0087`).

## Verified snippets

All snippets are from `DataMonitor.zip`. Corrected forms are marked.

**The gauge — `entry/src/main/ets/pages/Index.ets`** (corrected, see `HW-08-0083`)

```typescript
Refresh({ refreshing: $$this.isRefreshing }) {
  Stack() {
    Progress({
      // FIX: the sample's negative arm is a literal 100, so the ring fills
      // completely at the moment the allowance runs out.
      value: Math.max(0, Math.min(100,
        this.tempData * 100 / (this.dataNumber * Constants.GB_TO_MB))),
      type: ProgressType.Ring
    })
      .color(this.tempData > 0 ? $r('app.color.rest') : $r('app.color.used'))
      .backgroundColor($r('app.color.back'));

    Column() {
      Text(this.tempData.toFixed(0))          // MB remaining
      Text() { Span(this.time) }              // when it was read
    }
  }
}
.onRefreshing(async () => {                   // FIX: the sample does not await
  try {
    await this.tempDataMonitor();
  } finally {
    this.isRefreshing = false;                // FIX: sample uses setTimeout(500)
  }
})
```

**`Refresh` with `$$` two-way binding on `refreshing` is the correct pairing** -
the component sets the flag when the gesture starts and the app clears it when
the work is done. Clearing it on a timer instead breaks the contract in both
directions.

**`ProgressType.Ring` puts the number inside the gauge** via a `Stack`, which
is why the value and its label share a centre without any positioning.

**Storing the baseline — `entry/src/main/ets/utils/PreferencesClass.ets`** (as shipped)

```typescript
static savePreferenceInfo(content: Context, storeKey: string, storeName: string, value: number) {
  const STORE = PreferencesClass.getStore(content, storeName);
  STORE.putSync(storeKey, value);
  STORE.flushSync();                          // persist - putSync alone does not
}

static getPreferenceInfo(content: Context, storeKey: string, storeName: string) {
  const STORE = PreferencesClass.getStore(content, storeName);
  const VAL = STORE.getSync(storeKey, 0);     // numeric default, matching the stored type
  return VAL as number;
}
```

**`putSync` writes to the in-memory instance; `flushSync` writes it to disk.**
Omitting the flush is the single most common `preferences` mistake and this
sample avoids it on the save path - though not on the delete path
(`HW-08-0090`).

**The default is `0`, a number, matching what is stored.** `getSync` returns
the default when the stored value "is not of the default value type", so a
string default against numeric data would return the default forever - which is
exactly the defect in `KIDS-04`'s copy of this same helper class.

**Re-baselining — corrected, see `HW-08-0085`**

```typescript
async tempDataMonitor() {
  const rx = await statistics.getIfaceRxBytes(this.nic);
  const tx = await statistics.getIfaceTxBytes(this.nic);
  const total = rx + tx;

  // FIX: the sample subtracts unconditionally. Interface counters are not
  // monotonic across their own lifetime, and firstData is written once and
  // never revisited (`if (this.firstData === 0)`), so a lower reading yields a
  // negative usage and MORE remaining than the configured allowance.
  if (total < this.firstData) {
    this.firstData = total;
    PreferencesClass.savePreferenceInfo(this.context, this.flag,
      PreferencesClass.DEFAULT_STORE, this.firstData);
  }

  const used = Math.max(0, total - this.firstData);
  this.tempData = this.dataNumber * Constants.GB_TO_MB - used / Constants.BYTE_TO_MB;
  this.time = this.getTime();
}
```

## Permissions & config

**None, and that is correct.** `module.json5` declares no `requestPermissions`,
and the reference for `getIfaceRxBytes`/`getIfaceTxBytes` lists no required
permission - only a system capability,
`SystemCapability.Communication.NetManager.Core`. The per-application and
per-UID statistics APIs in the same module *do* need `ohos.permission.GET_NETWORK_STATS`;
the per-interface ones this sample uses do not.

`Constants.ets` holds the two conversion factors and nothing else:

```typescript
export class Constants {
  static readonly GB_TO_MB = 1024;
  static readonly BYTE_TO_MB = 1048576;
}
```

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **Only `wlan0` is measured** - the Wi-Fi interface, hardcoded at four call
  sites, which is not the traffic a data allowance is spent on
  (`HW-08-0084`).
- **The baseline can never be reset.** It is written only while it is exactly
  `0`, so after first launch there is no path that re-baselines - and a monthly
  allowance needs one every month.
- **The allowance is not persisted.** `dataNumber` is `@State` with a default of
  2 GB, so the parent's setting is lost when the page goes away, while the
  baseline it is compared against survives.
- **Readings are pull-only.** There is no background sampling, no notification
  when the quota is crossed and no history - the figure is whatever the last
  manual refresh saw.
- **Neither platform call is guarded.** No `try`/`catch` and no `.catch()`, so a
  missing interface rejects into nothing.
- The display is MB with no unit switch, so a 20 GB allowance reads as 20480.

## Pitfalls

- **`HW-08-0083` — the ring is driven to a full `100` when the allowance is
  exhausted.** The gauge shows *remaining* data, so it snaps from nearly empty
  to completely full at exactly the moment nothing is left - the strongest
  signal on the screen, inverted.
- **`HW-08-0084` — the interface is the literal `'wlan0'` at four call sites.**
  That is the WLAN interface, and Wi-Fi traffic is the traffic that does *not*
  consume a data allowance; no mobile interface is ever queried and nothing
  checks the name exists.
- **`HW-08-0085` — the baseline is written once and never revisited,** and
  `restData` is subtracted with no guard, so a reading below the stored baseline
  reports more remaining than the configured allowance and drives the ring past
  its maximum.
- **`HW-08-0086` — `onRefreshing` does not await `tempDataMonitor()`** and clears
  the spinner after a fixed 500 ms, so the indicator retracts before the reading
  arrives - and reports success for a refresh that rejected.
- **`HW-08-0087` — the allowance is `parseFloat(value)` with no validation,** so
  an empty field makes `dataNumber` `NaN` and every derived figure `NaN`,
  including the text in the middle of the ring.
- **`HW-08-0088` — `Context` is imported from `@ohos.abilityAccessCtrl`,** the
  permission module, in a sample that requests no permissions. The same file in
  `KIDS-04` has the same import.
- **`HW-08-0089` — `getTime()` round-trips a `Date` through its own
  milliseconds,** names the result `beijingTime` although no time zone is set,
  and hardcodes the `zh-CN` locale among otherwise resource-driven labels.
- **`HW-08-0090` — `deletePreferenceInfo` never flushes,** so the removal is not
  persisted; it is `async` with nothing awaited, and never called.

## References

- `documentation/harmonyos-references/03_system/js-apis-net-statistics.md` - `getIfaceRxBytes`, `getIfaceTxBytes`, the NIC parameter
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-net-statistics
- `documentation/harmonyos-references/02_application-framework/js-apis-data-preferences.md` - `getSync` and its default-type rule, `putSync`, `flushSync`, `deleteSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-data-preferences
- `documentation/harmonyos-references/02_application-framework/ts-container-refresh.md` - `Refresh` and `onRefreshing`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-refresh
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-progress.md` - `ProgressType.Ring` and `value`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-progress
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-textinput.md` - `inputType`, the missing restriction behind `HW-08-0087`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-textinput
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-sheet-transition.md` - the allowance sheet
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-sheet-transition
- `documentation/harmonyos-references/02_application-framework/js-apis-abilityaccessctrl.md` - the module `Context` was wrongly taken from
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-abilityaccessctrl
- `KIDS-04` - the other parental-monitoring sample, with the same `PreferencesClass` shape and a worse `getSync` default
- `KIDS-12` - the industry's other system-data sample
