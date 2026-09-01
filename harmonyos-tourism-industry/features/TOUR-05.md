---
id: TOUR-05
title: Origin and destination swap - animateTo slide plus a spinning swap icon
industry: 09_tourism
doc: huawei_industry_tree/09_tourism/docs/05_address_exchange.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/address_exchange-0000002283895117
sample: huawei_industry_tree/09_tourism/downloads/AddressExchange.zip
kits: ["@kit.ArkUI", "@ohos.curves"]
apis: ["UIContext.animateTo", "curves.springMotion", translate, rotate, animation, AnimateParam, CalendarPickerDialog, "UIContext.runScopedTask", "UIContext.getPromptAction", showToast, ForEach, Stack, Flex, FlexWrap]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-09-0028, HW-09-0029, HW-09-0030, HW-09-0031, HW-09-0032, HW-09-0033, HW-09-0080]
status: verified-with-fixes
---

## When to use

Load this card for the **swap control on a search form** - the round arrow
between "from" and "to" that reverses a journey. It appears in flight and
train booking, ride hailing, navigation, and anywhere a query has two ends
that a user might want the other way round.

The animation technique is the transferable part: two labels sliding past each
other under an explicit `animateTo`, with the icon spinning on its own
declarative `animation`. The same pair of ideas covers any "exchange these two
things" gesture - swapping currencies in a converter, two players in a lineup,
two panes in a comparison.

**Read `HW-09-0029` before adopting it.** The sample animates the positions
but never exchanges the values, which is fine for a screenshot and wrong for a
booking.

## Feature checklist

- A ticket search card over a hero image: origin on the left, destination on
  the right, a swap icon between them.
- Tapping the icon slides the two city names past each other and spins the
  icon.
- Tapping again slides them back.
- A departure date row that opens the system calendar picker and writes back
  the month, day and weekday.
- A search button and the date row raise a "demo only" toast.
- A history strip of previous origin/destination pairs; tapping one loads it
  into the form, respecting the current swap direction.

## Architecture

One `entry` module, one page and one view. No data layer beyond a static list.

```
entry/src/main/ets
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
├── model/HistorySearchModel.ets     HistoryClass + HISTORYLIST (5 static pairs)
├── pages/BuyingTickets.ets          @Entry, a Stack of the hero image + the view
└── view/AddressExchangeView.ets     the whole feature, 299 lines
```

The documented tree matches the zip.

`BuyingTickets` is twelve lines: a `Stack` aligned to `Alignment.Top` with a
background `Image` and `AddressExchangeView` over it. Everything else - state,
builders, animation, the picker, the history strip - lives in the one view
component, split into five `@Builder` methods (`startCity`, `endCity`,
`animatedIcons`, `historyRecord`, `dateShow`). For a component this size that
split is the right granularity: each builder is one visual block and the
`build()` body reads as a layout outline.

**The animation state is three numbers**: `translateX` (the slide offset),
`rotateAngle` (cumulative, never reset) and `swap` (the direction flag).
`translateX` is applied positively to the left label and negatively to the
right one, so a single value moves both halves symmetrically:

```
                translateX = 0                      translateX = 184.25
   [ 出发地            ][icon][         目的地 ]      [        出发地  ][icon][ 目的地       ]
     北京                          南京                        北京             南京
```

The offset is `rowWidth * 0.55`, derived from the container width rather than
hardcoded, which is the one piece of the geometry that survives a layout
change.

## Implementation steps

1. **Lay the row out symmetrically**: left label, icon, right label, each
   label `width('40%')` inside a fixed-width `Row`.
2. **Bind `translate` to a state number**, positive on the left label and
   negative on the right.
3. **Flip `textAlign` with the direction flag** so each label hugs the outer
   edge before the swap and the inner edge after it - without this the slide
   overshoots or leaves a gap.
4. **Animate the offset with `animateTo`** from `getUIContext()`, on
   `curves.springMotion()`. Include the else branch that returns the offset to
   zero (`HW-09-0030`).
5. **Exchange the two values**, not just their positions (`HW-09-0029`).
6. **Spin the icon declaratively**: `rotate({ angle })` plus `.animation({...})`
   on the same component, and increment the angle outside the `animateTo`
   closure so the two animations do not share a curve.
7. **Open the calendar picker inside `runScopedTask`** (`HW-09-0028`).
8. **Derive the displayed weekday from the date**, never store it separately
   (`HW-09-0031`).

## Verified snippets

All snippets are from `AddressExchange.zip`. Corrected forms are marked.

**The two labels — `entry/src/main/ets/view/AddressExchangeView.ets`** (as shipped)

```typescript
@Builder
startCity() {
  Column() {
    Text($r('app.string.start_address'))            // the fixed 出发地 label
      .alignSelf(ItemAlign.Start)
    Text(this.cityNameStart)
      .translate({ x: this.translateX })            // positive: slides right
      .textAlign(this.swap ? TextAlign.End : TextAlign.Start)
      .width('40%')
      .height(40)
  }
}

@Builder
endCity() {
  Column() {
    Text($r('app.string.end_address'))
      .alignSelf(ItemAlign.End)
    Text(this.cityNameEnd)
      .translate({ x: -this.translateX })           // negative: slides left by the same amount
      .textAlign(this.swap ? TextAlign.Start : TextAlign.End)
      .width('40%')
      .height(40)
  }
}
```

**Flipping `textAlign` with the direction is what makes one offset enough.**
Each city name lives in a 40%-wide box; before the swap both hug the outer
edges, after it both hug the inner ones. So the text travels the box width
plus the offset, and the two labels end up visually exchanged without needing
two different offsets or a measured layout.

Note that the fixed 出发地 / 目的地 labels do **not** move - only the city
values slide under them, which is precisely why the values must be exchanged
in state too.

**The swap handler — same file** (corrected, see `HW-09-0029`)

```typescript
import curves from '@ohos.curves';

private rowWidth: number = 335;
private distance: number = this.rowWidth * 0.55;   // offset derived from the container
private zeroTranslate: number = 0;
private rotateAddAngle: number = 360;

@Builder
animatedIcons() {
  Stack() {
    Image($r('app.media.change'))                   // the static arrows
    Image($r('app.media.circle'))                   // the ring that spins
      .rotate({ angle: this.rotateAngle })
      .animation({ curve: Curve.EaseOut, playMode: PlayMode.Normal })
  }
  .onClick(() => {
    this.swap = !this.swap;
    this.getUIContext()?.animateTo({ curve: curves.springMotion() }, () => {
      if (this.swap) {
        this.translateX = this.distance;
      } else {
        this.translateX = this.zeroTranslate;       // the else the document's snippet omits
      }
    });
    this.rotateAngle += this.rotateAddAngle;        // outside animateTo: its own curve

    // FIX: absent in the sample - swap the values, not only the pixels
    const from = this.cityNameStart;
    this.cityNameStart = this.cityNameEnd;
    this.cityNameEnd = from;
  });
}
```

**Two animation mechanisms, deliberately separate.** The slide runs inside an
explicit `animateTo` on `curves.springMotion()`, which gives the addresses a
physical settle. The ring runs on the component's own `.animation()` attribute
with `Curve.EaseOut`, so incrementing `rotateAngle` anywhere - including
outside the closure - animates it. Putting the angle inside the `animateTo`
closure, as the document's snippet does, would drag the spin onto the spring
curve instead (`HW-09-0030`).

`rotateAngle` is cumulative and never wrapped, which is correct: resetting it
to 0 would spin the ring backwards on the next press.

**Opening the calendar picker — same file** (corrected, see `HW-09-0028`)

```typescript
Text() {
  Span(`${this.monthDay}`)
  Span($r('app.string.month'))
  Span(`${this.day}`)
  Span($r('app.string.day'))
}
.onClick(() => {
  const uiContext = this.getUIContext();
  uiContext.runScopedTask(() => {              // FIX: the sample calls show() bare
    CalendarPickerDialog.show({
      selected: this.selectedDate,
      onAccept: (value) => {
        this.selectedDate = value;             // FIX: the sample never updates the seed
        this.weekDay = this.weekDays[value.getDay()];
        this.monthDay = value.getMonth() + 1;
        this.day = value.getDate();
      }
    });
  });
})
```

**`CalendarPickerDialog` is the one picker with no `UIContext` substitute.**
The UIContext guide's replacement table lists `AlertDialog`,
`DatePickerDialog`, `TimePickerDialog` and `TextPickerDialog` with their
`showXxx` equivalents, and `CalendarPickerDialog` with "Not supported" - so
`runScopedTask` is the sanctioned way to bind the bare global call to the
right UI instance.

Note also that `onAccept` in the sample updates the three display fields but
not `selectedDate`, so reopening the picker starts from the seed date again
rather than from the user's last choice.

**Wrapping history chips — same file** (as shipped)

```typescript
Flex({ justifyContent: FlexAlign.Start, wrap: FlexWrap.Wrap }) {
  if (HISTORYLIST.length > 0) {
    ForEach(HISTORYLIST, (item: HistoryClass, index: number) => {
      Text() {
        Span(item.startCityName)
        Span($r('app.string.split'))                 // the separator, itself a resource
        Span(item.endCityName)
      }
      .borderRadius(20)
      .backgroundColor('#F1F3F5')
      .fontColor(this.currentIndex === index ? '#0A59F7' : '#cc000000')
      .textOverflow({ overflow: TextOverflow.Ellipsis })
      .maxLines(2)
      .onClick(() => {
        this.currentIndex = index;
        this.cityNameStart = item.startCityName;     // FIX: the sample crosses these when swapped
        this.cityNameEnd = item.endCityName;
      })
    }, (item: HistoryClass, index: number) => index.toString())   // FIX: sample types item as string
  }
}
```

`Flex` with `wrap: FlexWrap.Wrap` is the right container for a chip strip of
unknown length - no grid maths, chips reflow at whatever width they need.
Composing the chip out of three `Span`s inside one `Text` keeps origin,
separator and destination on one baseline and lets a single
`textOverflow`/`maxLines` govern the whole chip.

Once the swap exchanges the real values (`HW-09-0029`), the crossed assignment
this handler needs today disappears.

## Permissions & config

**None.** The sample declares no `requestPermissions`.

Resource directories: `base`, `en_US`, `zh_CN`. The city names, labels and the
separator are all string resources - but `en_US/element/string.json` carries
only the three module boilerplate entries (`module_desc`,
`EntryAbility_desc`, `EntryAbility_label`), so nothing user-visible is
actually translated. The mechanism is right, the content is not filled in.

Numeric layout values live in `base/element/integer.json` and are referenced
as `$r('app.integer....')`, which is worth copying: the icon sizes, the date
row height and the button height are all tunable without touching the source.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- **`CalendarPickerDialog` throws on wearables.** The reference states: "On
  wearables, calling this API results in a runtime exception indicating that
  the API is undefined." Guard by device type if the app targets watches.
- The dialog "does not support dynamic updates for color mode changes" - it
  must be closed and reopened to pick up a light/dark switch.
- `rowWidth` is a hardcoded 335 vp and `distance` is derived from it, so the
  animation geometry is tied to one width. On a tablet or a resized 2in1
  window the labels slide the wrong distance. Measure the row instead if the
  form is responsive.
- The history list is five static entries in `HistorySearchModel.ets`; there
  is no persistence and no dedup.
- The search button and the date row are toasts - this is an animation sample,
  not a booking flow. For the date range see `TOUR-04`.

## Pitfalls

- **`HW-09-0029` — the swap moves pixels, not values.** `cityNameStart` still
  holds the original origin after a swap, so the label and the state disagree
  and any submit books the reverse trip. The crossed assignment in the history
  handler is the workaround that proves it.
- **`HW-09-0028` — `CalendarPickerDialog.show` is called bare.** It has no
  `UIContext` substitute, so the guide's prescribed form is
  `uiContext.runScopedTask(() => CalendarPickerDialog.show({...}))`.
- **`HW-09-0030` — the document's `animateTo` snippet is not the shipped
  code:** it moves the rotation inside the closure and omits the else branch,
  so a reader who copies it gets an offset that never returns to zero.
- **`HW-09-0031` — the page opens on 4月23日 周日 although 23 April 2025 is a
  Wednesday.** Three independent literals with no derivation. The seed
  `new Date('2025-04-23')` is also parsed as UTC, so it lands on the 22nd in
  western timezones.
- **`HW-09-0032` — the `ForEach` key generator declares `item: string`** for a
  `HistoryClass[]`, and keys on the stringified object, so two identical
  history pairs would collide.
- **`HW-09-0033` — `rotateAddAngle` is 360 under a comment saying 180.** A
  full turn returns the icon to its resting orientation, so it carries no
  indication of the current direction.

## References

- `documentation/harmonyos-references/02_application-framework/ts-explicit-animation.md` - `animateTo` and `AnimateParam`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-explicit-animation
- `documentation/harmonyos-references/02_application-framework/ts-animatorproperty.md` - the `animation` attribute
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-animatorproperty
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-transformation.md` - `translate` and `rotate`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-transformation
- `documentation/harmonyos-references/02_application-framework/ts-methods-calendarpicker-dialog.md` - `CalendarPickerDialog.show`, `CalendarDialogOptions`, the wearable exception
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-methods-calendarpicker-dialog
- `documentation/harmonyos-guides/03_application-framework/arkts-global-interface.md` - the global-API replacement table and `runScopedTask`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-global-interface
- `TOUR-04` - the date range picker that belongs in the same search form
- `TOUR-11` - the next step, confirming traveller details
