---
id: SOCIAL-01
title: Social & communication scenario index - the 44-sample map for the industry, and what it does not tell you
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/01_practice-socialcontact-app-architecture-v1_1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-socialcontact-app-architecture-v1_1-0000002267008353
sample: none
kits: []
apis: []
permissions: []
min_api: 20
modules: []
findings: [HW-14-0001, HW-14-0003]
status: verified
---

## When to use

**Load this card when you know you are building something social and do not yet
know which sample covers it.** 关键场景示例 (key scenario examples) is the index
page of Huawei's social & communication architecture guide: 44 links, no prose,
no sample project of its own. Every other card in this pack (`SOCIAL-02` to
`SOCIAL-45`) is one entry on it, in order.

Use it as a routing table, not as a source. It answers "does an official sample
exist for X" and nothing else - there is no architecture narrative, no
recommended layering, no shared component library, and no statement of which
samples are meant to compose. Each linked scenario is an independent single-`entry`
demo, and several of them ship the *same* feature twice from different templates
(three separate voice-to-text pages, two image-preview viewers, two encryption
demos).

Because it is an index and not a document, this card's verification level is
low: nothing here was compiled or run. What was verified is the *corpus* the
index points at - all 44 zips were downloaded and read, and the two systematic
defects below are the cross-cutting results.

## Feature checklist

What the page promises:

- A flat, ordered list of 44 key social & communication scenarios.
- Each entry links to an architecture-guide page with a scenario description, an
  implementation narrative, a constraints block, a project tree and a zipped
  sample.
- Coverage of the recognisable surfaces of a messaging app: the chat page, the
  chat list, the friend feed, the contact book, media handling and transport.

What it does not provide, and you should not go looking for:

- No shared module, design system, or common utility package across the samples.
- No dependency or ordering between scenarios.
- No statement of which samples supersede which - duplicates are simply both
  listed.

## Architecture

The page is a single unordered list. The samples behind it cluster into six
groups, which is the useful way to read it:

```
Chat page surface        SOCIAL-03 voice→text, 23 voice→text (2nd take), 33 multi-select+forward,
                         14 quote & jump, 13 unread badges, 27 chat bubbles, 40 fold & pin,
                         36 quick reply (LazyForEach), 37 emoji reactions, 25 emoji recommend,
                         35 back-to-latest, 30 inertial scrolling of a long image
Media in chat            SOCIAL-02 image preview, 08 send-original toggle, 29 custom album picker,
                         39 recent images, 34 drag-to-reorder a 9-grid, 16 drag-and-drop send,
                         41 receive a share from another app, 10 avatar upload
Identity & contacts      SOCIAL-09 contact list, 15 add friend via contact, 42 send a contact card,
                         38 personalised QR code, 24 long-press QR scan, 31 phone-number detection
Discovery & feed         SOCIAL-04 friend match + loading animation, 05 image recognition,
                         06 nearby-people distance, 07 vote results, 18 draft on exit,
                         19 group solitaire, 28 app message list
Security & web           SOCIAL-11 chat encryption, 43 file encryption, 12 URL interception,
                         22 history web float
Transport & platform     SOCIAL-20 weak-network IM, 44 WebSocket client, 17 Bluetooth SPP,
                         21 voice call page, 26 location card + navigation, 32 chat-history search,
                         45 one-build multi-device navigation design
```

(Numbers are this pack's `SOCIAL-NN` ids, which follow the index order.)

**The design decision worth noting - and avoiding - is that there isn't one.**
Every sample is a standalone `entry` module built from the same DevEco project
template, and the template is where the two systematic defects come from. Read
positively: you can lift any one sample without pulling in a framework. Read
honestly: forty-four copies of `EntryAbility.ets` differ only in the page they
load, and forty-four copies of `module.json5` inherit the same permission block
whether or not the feature uses it (`HW-14-0003`). Nothing in this index warns
you about either.

## Implementation steps

1. **Find the scenario by cluster**, not by scanning the 44-item list - the page
   is ordered by publication, not by subject.
2. **Open the matching `SOCIAL-NN` card first.** Each one carries the zip-verified
   architecture, the corrected snippets and the defect list; the upstream page
   carries neither.
3. **Verify the 工程目录 block against the zip before navigating by it.** Four of
   this industry's trees list files that do not exist (`HW-14-0001`).
4. **Audit `module.json5` against actual call sites** whenever you lift a sample
   as a starting point - the template's permission block travels with it
   (`HW-14-0003`).
5. **Check for a duplicate before committing to one sample.** Where two samples
   cover the same feature, they were written independently and are not equally
   good: `SOCIAL-03` and `SOCIAL-23` both do voice-to-text, `SOCIAL-02` and
   `SOCIAL-08` share an image viewer (and share `HW-14-0004` with it).
6. **Do not expect the samples to compose.** Two of them dropped into one app
   will collide on `AppStorage` keys, on `main_pages`, and - in the case of the
   `RichEditor` samples - on a module-level controller singleton.

## Verified snippets

(from the doc - no sample shipped; not compile-verified)

The page's entire body is link markup. The first three entries, which are the
three zip-backed cards rewritten alongside this one:

```markdown
- **[好友动态-图片预览](https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_preview-0000002266277321)**
- **[聊天页-语音输入文字](https://developer.huawei.com/consumer/cn/doc/architecture-guides/voice_to_text_forchat-0000002231520016)**
- **[好友推荐列表排版和加载动画](https://developer.huawei.com/consumer/cn/doc/architecture-guides/preference_search-0000002236253766)**
```

好友动态-图片预览 is "friend feed - image preview" (`SOCIAL-02`);
聊天页-语音输入文字 is "chat page - voice input to text" (`SOCIAL-03`);
好友推荐列表排版和加载动画 is "friend recommendation list layout and loading
animation" (`SOCIAL-04`). The Chinese titles are the only description the index
gives, which is why the `SOCIAL-NN` card titles in this pack are rewritten as
English statements of the *pattern* rather than translations of the label.

Every linked page shares one fixed section skeleton, and knowing it is what
makes the corpus scannable:

```markdown
## 场景介绍     scenario introduction - one paragraph, plus the APIs it is built on
## 效果预览     a single animated gif
## 实现思路     implementation approach - the numbered narrative, with code excerpts
## 约束与限制   constraints - always the same three SDK/IDE version lines
## 权限说明     permissions - present only when the sample declares any
## 工程目录     project tree - the block that is not always accurate
## 参考文档     reference links
## 代码下载     the sample zip
```

The 实现思路 excerpts are **retyped, not extracted**: across this corpus they
routinely differ from the shipped code in ways that matter - dropped `else`
branches, missing guards, reordered statements. Every snippet in the other cards
in this pack is taken from the zip for that reason.

## Pitfalls

- **`HW-14-0001`** (E/low, confirmed): four of the linked pages document a
  project tree the zip does not match - `05_image_recognition` lists
  `Entryability.ets` where the zip has `EntryAbility.ets`;
  `29_custom_album_style` lists an absent `PhotoPickerView.ets`;
  `36_quick_reply_in_chat` lists an absent `DataUtil.ets`; and
  `44_web_socket_client2` lists `pages/Index.ets` where the zip's only page is
  `WebSocketClient.ets`. Trees were not regenerated after renames. Fix: read the
  zip listing, not the 工程目录 block.
- **`HW-14-0003`** (D/low, confirmed): one copy-pasted permission template runs
  through the corpus. `33_chat_multi_selection` declares `INTERNET` and `VIBRATE`
  with no code using either; `42_chat_send_conntact` declares an unused
  `INTERNET`; `18_save_draft_on_exit` and `34_drag_image_sort` ship a
  `CommonConstants.REQUEST_PERMISSIONS` array containing `LOCATION` /
  `APPROXIMATELY_LOCATION` that nothing ever passes to
  `requestPermissionsFromUser` - location has no role in a chat-draft or an
  image-sort demo. Excess declarations break least privilege and dead constants
  invite someone to wire them up. Fix: delete both.
- **The index has no quality signal.** Duplicated scenarios, samples whose
  headline feature is inert (`SOCIAL-04`, `HW-14-0009`), and samples with
  user-facing defects on the primary path (`SOCIAL-03`, `HW-14-0005`/`0006`) are
  listed identically to clean ones. Treat every entry as a starting point to be
  reviewed, not as reference code.
- **Ordering is publication order.** Neighbouring entries are unrelated, and a
  later entry may be a better implementation of an earlier one. There is no
  deprecation marker anywhere on the page.
- The upstream index is versioned `practice-socialcontact-app-architecture-v1_1`.
  A `v1_2` will renumber nothing but may append; the `SOCIAL-NN` ids in this pack
  are pinned to the v1_1 order.

## References

- `huawei_industry_tree/14_social_communication/docs/01_practice-socialcontact-app-architecture-v1_1.md` - the index page itself
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-socialcontact-app-architecture-v1_1-0000002267008353
- `SOCIAL-02` - friend feed image preview (zip-verified)
- `SOCIAL-03` - chat page voice input to text (zip-verified)
- `SOCIAL-04` - friend match layout and loading animation (zip-verified)
- `SOCIAL-05` - carries `HW-14-0001`, the tree-mismatch systematic
- `SOCIAL-33` - carries `HW-14-0003`, the permission-template systematic
- `documentation/harmonyos-guides/04_system/request-user-authorization.md` - the permission flow every sample in this corpus reimplements
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/request-user-authorization
