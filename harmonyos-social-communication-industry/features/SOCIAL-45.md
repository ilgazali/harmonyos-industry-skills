---
id: SOCIAL-45
title: Industry FAQ redirect - the page that answers nothing, and where the answers actually are
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/45_practice-socialcontact-app-architecture-v1_2.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-socialcontact-app-architecture-v1_2-0000002298361245
sample: none
kits: []
apis: []
permissions: []
min_api: 20
modules: []
findings: []
status: verified
---

## When to use

**Load this card when you went looking for the social & communication industry
FAQ and want to know whether it is worth the click.** 行业常见问题 (industry
frequently asked questions) is the last page of Huawei's social architecture
guide, and it is one sentence long: the content has moved to the general
HarmonyOS FAQ site. There is no social-specific material behind the link, no
sample project, and no anchor that lands on a communication section.

So the practical answer is: it is not worth the click for anything covered by
this pack. The general FAQ is organised by device and by framework subsystem,
not by industry, and it answers "why does my build fail" far more often than
"how should a chat page be structured". This card exists to close that loop -
to say plainly what the page is, and to route the questions it *sounds* like it
would answer to the sibling cards that actually answer them.

Its verification level is the lowest in the pack. Nothing was compiled, nothing
was run, and the linked destination is outside the corpus we reviewed. What
*is* verified is the routing below: every card named here was written against a
sample zip we read, and every `HW-…` id cited was raised and confirmed during
that review.

## Feature checklist

What the page promises, in full:

- A pointer to the HarmonyOS FAQ portal (`harmonyos-faqs/faq-phone`).

That is the entire content. For completeness, what it does **not** provide,
despite the title:

- No social-specific questions or answers.
- No troubleshooting for any of the 43 samples the guide links.
- No sample project, no code, no configuration.
- No deep link into a communication-related FAQ section.

## Architecture

Documentation-only page, no sample project. Eleven lines of markdown in the
mirror, of which one is prose:

```
huawei_industry_tree/14_social_communication/docs
├── 01_practice-socialcontact-app-architecture-v1_1.md   the 44-link scenario index -> SOCIAL-01
├── 02_image_preview.md … 44_web_socket_client2.md       the scenario pages -> SOCIAL-02 … SOCIAL-44
└── 45_practice-socialcontact-app-architecture-v1_2.md   this page: a redirect
```

**The design decision worth understanding** is structural rather than technical:
the industry guide is a two-page frame (`SOCIAL-01` index, `SOCIAL-45` FAQ)
around 43 independent scenario demos. There is no layer between them. No
shared component library, no recommended module decomposition, no statement of
which samples compose - and now no industry FAQ either, because it was folded
into the platform-wide one.

That is worth knowing before you plan work against this guide: it is a catalogue
of solved scenarios, not an architecture. Anything cross-cutting - a message
model, a bubble component, a connection layer, a permission policy - you own.
The recurring defects our review found are largely a consequence of that shape:
43 teams each cloned a chat page template, and the same handful of mistakes
travelled with it.

## Implementation steps

There is nothing to implement. What to do instead, in order:

1. **Start from `SOCIAL-01`,** the scenario index, if you do not yet know which
   sample covers your feature.
2. **Check the routing table below** before opening the external FAQ - most
   questions that sound like FAQ material are answered by a specific card here,
   with the sample source read and the defects named.
3. **Assume nothing composes.** Each sample is a standalone single-`entry`
   demo; taking two of them means reconciling two message models.
4. **Carry the systematic findings into your own code.** They are not
   sample-specific: `HW-14-0018` (`ForEach` keys), `HW-14-0031` (`isSelf`
   toggled before send), `HW-14-0003` (copy-pasted permissions), `HW-14-0002`
   (cleartext `ws://`) and `HW-14-0001` (documented trees that do not match the
   zips) each hit several projects at once.
5. **Only then go to the platform FAQ,** for toolchain, signing, device and
   emulator questions, which is what it is actually good for.

## Verified snippets

**The whole page — `huawei_industry_tree/14_social_communication/docs/45_practice-socialcontact-app-architecture-v1_2.md`**
(from the doc — no sample shipped; not compile-verified)

```markdown
# 行业常见问题

行业常见问题内容已迁移至HarmonyOS FAQ，请点击[此处](https://developer.huawei.com/consumer/cn/doc/harmonyos-faqs/faq-phone)前往。
```

"The industry FAQ content has been migrated to the HarmonyOS FAQ; please click
here to go there." The link target is `faq-phone` - the phone FAQ index, not a
social section - so a reader arrives at a general troubleshooting portal with
no filter applied and no memory of the industry they came from.

**Where the questions this page implies are actually answered** (routing, verified
against the cards in this pack)

```
"How do I receive a share from another app?"        -> SOCIAL-41  (skills + getSharedData)
"How do I send a file / an encrypted file?"         -> SOCIAL-43  (isEncryptionSupported)
"How do I send a contact card / a location?"        -> SOCIAL-42, SOCIAL-26
"How do I keep a socket open for messaging?"        -> SOCIAL-44, SOCIAL-20
"Why does my second identical message not render?"  -> HW-14-0018 (SOCIAL-08)
"Why is my first message on the wrong side?"        -> HW-14-0031 (SOCIAL-14, SOCIAL-42)
"Which permissions does a chat app really need?"    -> HW-14-0003 (SOCIAL-33)
"Which sample covers X at all?"                     -> SOCIAL-01  (the index)
```

The four scenario cards in that table were each written from the extracted zip
rather than the document, which is the difference in evidence level between them
and this page.

## Pitfalls

**No findings were raised against this page** - there is no code to be wrong
and one sentence of prose, which is accurate. The problems are editorial, and
worth stating because they change how much you should trust the guide:

- **A redirect is not an answer.** Shipping an industry guide whose FAQ chapter
  points at a device-level portal means the industry has no FAQ. Questions
  specific to chat - message ordering, attachment lifetime, reconnect policy -
  have no documented home.
- **The link is unanchored.** `faq-phone` is an index. Nothing carries the
  reader's industry context across, so the migration lost the categorisation
  that made the page findable in the first place.
- **This page and `SOCIAL-01` are the only two non-scenario pages in the
  guide,** and neither contains architecture. A reader looking for the
  "architecture" the guide's title promises will not find it in either.
- **Verification here is documentation-level only.** Unlike every other card in
  this pack, nothing behind this one was read as source. Treat its claims as
  claims about the *page*, not about the platform.

## References

- `huawei_industry_tree/14_social_communication/docs/45_practice-socialcontact-app-architecture-v1_2.md` - the page itself
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-socialcontact-app-architecture-v1_2-0000002298361245
- `SOCIAL-01` - the scenario index, the other half of the guide's frame, and the
  card that carries the corpus-wide findings `HW-14-0001` and `HW-14-0003`
- `SOCIAL-08` - the origin of `HW-14-0018`, the systematic `ForEach` key defect
- `SOCIAL-14` - the origin of `HW-14-0031`, the `isSelf`-before-send defect
- `SOCIAL-20` - the origin of `HW-14-0002`, cleartext `ws://` endpoints
- `SOCIAL-41`, `SOCIAL-42`, `SOCIAL-43`, `SOCIAL-44` - the four scenarios that
  close the guide, all zip-verified
