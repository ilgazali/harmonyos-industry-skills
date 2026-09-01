---
id: KIDS-08
title: Poem annotation template - tap an underlined word for a popup gloss
industry: 08_children_education
doc: huawei_industry_tree/08_children_education/docs/08_poetry_analysis_template.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/poetry_analysis_template-0000002319599640
sample: huawei_industry_tree/08_children_education/downloads/PoetryTemplate.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit"]
apis: [bindPopup, PopupOptions, onStateChange, messageOptions, BlurStyle, Text, letterSpacing, Divider, ForEach, "@State", expandSafeArea, SafeAreaType, SafeAreaEdge, "UIAbilityContext.terminateSelf", alignRules]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-08-0061, HW-08-0062, HW-08-0063, HW-08-0064, HW-08-0065, HW-08-0066, HW-08-0067, HW-08-0120]
status: verified-with-fixes
---

## When to use

Load this card for **annotated reading**: text where selected words carry an
explanation that appears on tap. A classical poem here, and equally a foreign
language reader, a glossary in a textbook, a legal or medical document with
defined terms.

The mechanism is `bindPopup` driven by a per-word boolean, and two things make
it work:

- **The text is data, not markup.** A line is an array of fragments, each with
  `desc` (the text), `comment` (the gloss) and `id` (0 = plain, 1 =
  annotated). Rendering is a nested `ForEach` over lines and fragments, so the
  poem can be swapped without touching the layout.
- **One open popup at a time, by construction.** Every tap first walks the
  whole model setting `display = false`, then sets the one it wants. There is
  no "currently open" pointer to get out of step.

**Seven findings.** None is fatal, but two are worth knowing before copying:
eight `alignRules` calls do nothing because there is no `RelativeContainer`,
and the page writes its popup flags straight into the imported JSON module.

## Feature checklist

- A poem with its title, dynasty and author.
- Annotated words carry an orange underline; plain words do not.
- Tapping an annotated word opens a popup with its gloss.
- Opening one annotation closes any other.
- Tapping outside dismisses the popup.
- A back button closes the app.

## Architecture

One `entry` module, one page, one JSON data file. 383 lines total.

```
entry/src/main/ets
├── entryability/EntryAbility.ets   loadContent only
├── entrybackupability/
└── pages
    ├── Index.ets                   the whole page + four interfaces
    └── data.json                   title, name, content
```

The documented tree matches the zip.

**The data model is three parallel shapes** with the same annotation fields:

```typescript
interface ContentValue {
  'width': number,    // 注释线宽度  - underline length, in characters
  'display': boolean, // 控制弹框显隐 - is this word's popup open
  'desc': string,     //              the word itself
  'num': number,      // 控制弹框宽度 - gloss length, drives popup width
  'comment': string,  // 解析         the gloss
  'id': number        // 是否有注释 0 没有，1 有
}
```

`Title` and `Author` carry the same six fields under different names. Keeping
the annotation state (`display`) on the same object as the content is what
makes the "close everything, open one" pass a simple walk.

**The render is a two-level `ForEach` with a branch per fragment:**

```
content[0]  ──ForEach(line)──>  Row
                                 └─ForEach(fragment)──> id === 0 ? plain Text
                                                                 : Column { Text + Divider }
```

Only annotated fragments get the `Column` wrapper, because only they need an
underline beneath the text. Plain fragments are a bare `Text`, so they cost
nothing extra.

**The underline is a `Divider`, not a text decoration.** That is deliberate:
`decoration({ type: TextDecorationType.Underline })` would underline the whole
`Text` in the text colour, and this design wants a separate orange rule whose
width it controls. The cost is that the width has to be computed
(`HW-08-0065`).

## Implementation steps

1. **Model each line as an array of fragments**, annotated or not.
2. **Render annotated fragments as `Text` + `Divider`** inside a `Column`.
3. **Bind a popup to each annotated `Text`**, keyed on that fragment's
   `display`.
4. **On tap, clear every flag, then set one** - and extract that reset
   (`HW-08-0067`).
5. **Clear the flag in `onStateChange`** so an outside tap leaves the model
   consistent.
6. **Copy the JSON on entry** rather than mutating the module (`HW-08-0062`).
7. **Measure the word to size the underline** instead of counting characters
   (`HW-08-0065`).

## Verified snippets

All snippets are from `PoetryTemplate.zip`. Corrected forms are marked.

**The annotated fragment — `entry/src/main/ets/pages/Index.ets`** (corrected, see `HW-08-0063`, `HW-08-0066`, `HW-08-0067`)

```typescript
ForEach(this.testValue, (item: Value, index) => {
  Row() {
    ForEach(item.content, (contentValue: ContentValue, idx) => {
      if (contentValue.id === 0) {
        Text(contentValue.desc)                 // plain fragment, no wrapper
          .fontSize(18)
          .letterSpacing(20)
      } else {
        Column() {
          Text(contentValue.desc)
            .fontSize(18)
            .letterSpacing(20)
            .onClick(() => {
              this.description = this.testValue[index].content[idx].comment
              this.closeAllPopups()             // FIX: the sample inlines this loop
              this.testValue[index].content[idx].display = true
            })
            .bindPopup(this.testValue[index].content[idx].display, {
              message: this.description,
              mask: false,                      // the page stays interactive
              radius: 14,
              popupColor: Color.White,
              width: contentValue.num > 15 ? 220 : (contentValue.num + 2) * 14,
              offset: { x: -15, y: 0 },
              backgroundBlurStyle: BlurStyle.NONE,
              messageOptions: { textColor: Color.Black, font: { size: '14vp' } },
              autoCancel: false,
              onStateChange: (e) => {
                if (!e.isVisible) {
                  this.closeAllPopups()         // keep the model in step
                }
              },
            })
          Divider().color(Color.Orange).strokeWidth(1)
            .width(this.measureWord(contentValue.desc))   // FIX: see HW-08-0065
            .margin({ top: 2 })
        }
        .alignItems(HorizontalAlign.Start)
      }
    })
  }
})
```

**`onStateChange` is what keeps the flag honest.** `bindPopup` closes itself on
an outside tap without telling the model, so without this callback `display`
would stay `true` and the next tap on that word would be a no-op. Any
`bindPopup` driven by your own boolean needs this pairing.

**`mask: false` matters for a reading app** - the default dim mask would block
taps on the rest of the poem, so a child could not move from one annotation to
the next in a single tap.

**`autoCancel: false` with `onStateChange` looks contradictory and is not.**
`autoCancel` governs whether a tap outside dismisses the popup;
`onStateChange` fires whenever visibility changes for any reason. The
combination here means the popup closes only when another word is tapped or
the flag is cleared.

**A single shared `description` field feeds every popup.** All the
`bindPopup`s read `this.description`, and the tap handler sets it before
opening. That works because only one popup is ever visible, and it avoids
storing the resolved gloss on every fragment.

**Closing everything, then opening one — corrected**

```typescript
// FIX: the sample repeats this nested loop inline in four places, and the
// fourth copy omits the title and author lines.
closeAllPopups(): void {
  this.testValue.forEach((item) => {
    item.content.forEach((content) => {
      content.display = false
    })
  })
  this.author.display = false
  this.title.display = false
}
```

**The title and author handlers — same file** (corrected, see `HW-08-0063`)

```typescript
.onClick(() => {
  this.description = this.title.comment
  // FIX: the sample ends with
  //   this.title.display = false
  //   this.title.display = !this.title.display
  // The reset on the line above already cleared it, so the negation is
  // unconditionally true and a second tap cannot close the popup.
  const wasOpen = this.title.display
  this.closeAllPopups()
  this.title.display = !wasOpen
})
```

**Loading the data — corrected, see `HW-08-0062`**

```typescript
import data from './data.json'

@State title: Title = data.title[0]        // FIX: these are references INTO the
@State author: Author = data.name[0]       // module object, and every handler
@State testValue: Value[] = data.content[0] // writes display flags through them

// FIX: copy on entry so the module stays immutable
aboutToAppear(): void {
  this.title = JSON.parse(JSON.stringify(data.title[0]))
  this.author = JSON.parse(JSON.stringify(data.name[0]))
  this.testValue = JSON.parse(JSON.stringify(data.content[0]))
}
```

**A module object is created once per process.** `@State` on a reference into
it observes the reference, not the fields, so the decorators suggest ownership
the component does not have - and the writes land in data shared by every
importer.

**Sizing the underline — corrected, see `HW-08-0065`**

```typescript
// FIX: the sample computes three different character-count multipliers -
//   title:   this.title.width * 22
//   author:  this.author.width * 18
//   body:    contentValue.width > 1 ? (contentValue.width * 2 - 1) * 18
//                                   : contentValue.width * 18
// The body correction exists only to undo letterSpacing(20).
measureWord(word: string): number {
  const size = this.getUIContext().getMeasureUtils().measureTextSize({
    textContent: word,
    fontSize: '18vp',
    letterSpacing: 20
  })
  return this.getUIContext().px2vp(Number(size.width))
}
```

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`. No network, no
storage - the poem is a bundled JSON file.

`main_pages.json` lists `pages/Index` alone; there is no navigation, which is
why the back button terminates the ability instead of popping:

```typescript
.onClick(() => {
  let context = this.getUIContext().getHostContext() as common.UIAbilityContext
  context.terminateSelf()
})
```

`EntryAbility` is the thinnest in the industry - `loadContent` and nothing
else. No full-screen call, no avoid-area handling, which is what makes
`HW-08-0064` a problem: the page opts into the safe areas with
`expandSafeArea` and there is no inset available to pad with.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **One poem.** `data.json` holds a single title, a single author and a single
  `content[0]`; the arrays exist but nothing selects among them and there is no
  poem list or navigation.
- **The layout is tuned for short lines.** `letterSpacing(20)` at 18 points
  with a 90%-wide card fits a five-character classical line; a longer line
  wraps inside its `Row` and the underlines separate from the text.
- **`width` and `num` are hand-maintained per fragment** in the data file - the
  character count for the underline and the gloss length for the popup width.
  Adding an annotation means counting characters by hand.
- **The popup width caps at 220** for any gloss over 15 characters, so the
  author's 65-character biography is squeezed into a fixed box.
- **No text selection, no search, no audio** - tapping is the only interaction.

## Pitfalls

- **`HW-08-0061` — eight `alignRules` calls sit on children of `Column` and
  `Row`,** and the reference states the attribute "only takes effect when the
  parent container is `RelativeContainer`". There is no `RelativeContainer` in
  the file; the centring that is visible comes from the `Column` defaults.
- **`HW-08-0062` — the page writes `display` flags into the imported
  `data.json` module,** so annotation state is process-global and survives the
  page being destroyed. The body handler reassigns `this.testValue =
  data.content[0]` on every tap, which is only meaningful because they are the
  same object.
- **`HW-08-0063` — the title and author "toggles" are unconditionally true:**
  `display = false` on one line, `display = !display` on the next. Tapping the
  same word twice reopens the popup instead of closing it.
- **`HW-08-0064` — the page calls `expandSafeArea` for both edges and pads for
  neither,** and the ability reads no avoid area at all, so the back button and
  title are drawn under the status bar.
- **`HW-08-0065` — underline and popup widths are character counts times
  hand-fitted constants** (×22, ×18, ×14, plus a `width * 2 - 1` correction for
  `letterSpacing`), so both drift from the text under system font scaling.
  `measureTextSize` exists and `KIDS-05` uses it.
- **`HW-08-0066` — `.id(contentValue.desc)` takes the component id from the
  displayed word,** where the reference requires uniqueness "guaranteed by the
  user" - and repetition is ordinary in the classical verse this template is
  for.
- **`HW-08-0067` — the four-line reset loop is copied into four handlers,** and
  the fourth copy already differs: it clears the body flags but not the title
  and author ones.

## References

- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-popup.md` - `bindPopup`, `PopupOptions`, `onStateChange`, `autoCancel`, `mask`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-popup
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-text.md` - `Text` and `letterSpacing`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-text
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-location.md` - `alignRules` and its `RelativeContainer` requirement
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-location
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-component-id.md` - `id` and the uniqueness contract
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-component-id
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-expand-safe-area.md` - `expandSafeArea`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-expand-safe-area
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-measureutils.md` - `measureTextSize`, the fix for `HW-08-0065`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-measureutils
- `documentation/harmonyos-references/02_application-framework/ts-container-relativecontainer.md` - the container `alignRules` needs
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-relativecontainer
- `KIDS-13` - the other reading sample in this industry, with audio instead of glosses
- `KIDS-05` - the sample that measures text properly
