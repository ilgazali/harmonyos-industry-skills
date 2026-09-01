---
id: SHOP-24
title: Shopping industry FAQ node - a redirect page, and where its questions actually get answered
industry: 16_shopping
doc: huawei_industry_tree/16_shopping/docs/24_practice-purchase-app-architecture-v1_2.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-purchase-app-architecture-v1_2-0000002298474325
sample: none
kits: []
apis: []
permissions: []
min_api: n/a
modules: []
findings: []
status: verified
---

## When to use

Load this card when something points you at the shopping industry's 行业常见问题
(industry FAQ) page and you need to know **what is actually there**. The
answer is: nothing. The page's entire body is one sentence saying the content
has moved to the HarmonyOS FAQ portal, plus the link. There is no sample, no
snippet, no constraint list, and nothing to verify.

The card exists so that a dead end is a *documented* dead end. Its useful
content is the routing below: the questions this node used to absorb are now
answered either by the general FAQ portal (device, signing, IDE, account and
account-level questions) or by one of the twenty-one scenario cards in this
pack (anything about how a shopping screen is actually built). Knowing which
of the two applies saves a round trip through a page that says nothing.

Treat this as the third and last node of the industry's architecture series:
`SHOP-01` is the architecture narrative, `SHOP-02` is the index of scenario
samples, and `SHOP-24` is the FAQ pointer. All three are documentation nodes;
only `SHOP-01` carries a sample.

## Feature checklist

What the page promises, in full:

- A single link to `harmonyos-faqs/faq-phone` under the label 此处 (here).
- No scenario content, no code, no project tree, no constraints section.

Verified against the crawled copy at
`huawei_industry_tree/16_shopping/docs/24_practice-purchase-app-architecture-v1_2.md`:
the crawl captured the page in full - eleven lines including frontmatter - and
nothing was truncated. The `images: 0` frontmatter field is correct; the page
has none.

## Architecture

No project, no zip, no module. The node's only structural role is its position
in the industry tree:

```
16_shopping (architecture-guides)
├── 01 购物商城应用案例                practice-purchase-app-architecture-v1    -> SHOP-01 (MultiShopping.zip)
├── 02 关键场景示例                    practice-purchase-app-architecture-v1_1  -> SHOP-02 (index page, 21 links)
│   ├── 03..20  the twenty scenario samples listed by 02
│   ├── 21 商品对比查询                goods_pk                                 -> SHOP-21 (GoodsPK.zip)
│   ├── 22 首页下拉进入二楼            second_floor                             -> SHOP-22 (SecondFloor.zip)
│   └── 23 地址列表与地址管理          address_manager                          -> SHOP-23 (AddressManager.zip)
└── 24 行业常见问题                    practice-purchase-app-architecture-v1_2  -> this card (redirect only)
```

**The design decision worth noting** - and it is an editorial one, not a code
one - is that the FAQ was pulled out of the industry tree and centralised.
That is defensible: a per-industry FAQ inevitably restates platform-level
answers (signing, device compatibility, DevEco versions, account setup) that
have nothing to do with shopping, and keeping twenty of them in step is
hopeless. The cost is that the industry tree now has a node that resolves to
a general portal with no shopping filter on it, so a reader arriving with a
shopping question lands on a page about phones in general.

The practical consequence for this skill pack: **the FAQ node answers no
scenario question**, and questions that look like FAQ material should be
routed to a scenario card instead. That routing is the next section.

## Implementation steps

There is nothing to implement. What follows is the routing to use when a
question arrives at this node.

1. **Platform-level questions go to the FAQ portal**: signing and
   provisioning, device and emulator setup, DevEco Studio versions, SDK
   installation, HarmonyOS account and AGC configuration, publishing. None of
   these are industry-specific and none are covered by this pack.
2. **"How do I build X screen" goes to a scenario card.** The twenty-one
   scenario cards in this pack are the industry's real answer surface:

   | Question shape | Card |
   | --- | --- |
   | Whole-app skeleton, responsive home, cart, orders, scan | `SHOP-01` |
   | Coupon wallet, claiming, duplicate-claim guard | `SHOP-03`, `SHOP-05` |
   | Check-in, points, browse tasks | `SHOP-04` |
   | Pull-down gestures: jump to a page / open a second floor | `SHOP-06`, `SHOP-22` |
   | Lottery, scratch card, red-envelope rain | `SHOP-07`, `SHOP-17` |
   | Product cards: long-press marking, comparison | `SHOP-08`, `SHOP-21` |
   | Search: history, expand/collapse, recommendation carousel | `SHOP-09`, `SHOP-12`, `SHOP-20` |
   | Feed behaviour: sticky headers, skeleton screens, auto-height menus | `SHOP-10`, `SHOP-11`, `SHOP-14` |
   | Orders: status switching, post-delivery review | `SHOP-13` |
   | Local storage: browsing history, addresses | `SHOP-15`, `SHOP-23` |
   | Timers and reminders: ticket-grab countdown | `SHOP-16` |
   | Badges, unread counts, live-updating numbers | `SHOP-18`, `SHOP-19` |
3. **Architecture questions go to `SHOP-01`** - module division, the
   HAP/HAR layering, which features belong in which package.
4. **"Is this snippet correct?" - assume not, and check the zip.** Across
   this industry's documents the published excerpts are abridged in ways that
   break them (`HW-16-0013`). Every scenario card in this pack quotes the
   extracted sample source instead, which is why the cards and the documents
   sometimes disagree.

## Verified snippets

There is no sample. The page's complete body, from the doc
(from the doc — no sample shipped; not compile-verified):

```markdown
# 行业常见问题

行业常见问题内容已迁移至HarmonyOS FAQ，请点击[此处](https://developer.huawei.com/consumer/cn/doc/harmonyos-faqs/faq-phone)前往。
```

That is the entire page. 行业常见问题 is "industry frequently asked
questions"; the sentence reads "the industry FAQ content has been migrated to
the HarmonyOS FAQ, please click *here* to go there." The link target is the
general phone FAQ, not a shopping-filtered view - there is no shopping anchor
on it.

For comparison, the sibling node that *does* carry content is `SHOP-02`,
whose body is a flat list of the twenty-one scenario links:

```markdown
- **[优惠券卡包](https://developer.huawei.com/consumer/cn/doc/architecture-guides/coupons_page-0000002235638646)**
- **[签到、完成浏览任务获取积分](https://developer.huawei.com/consumer/cn/doc/architecture-guides/sign-0000002284183885)**
- ...
- **[商品对比查询](https://developer.huawei.com/consumer/cn/doc/architecture-guides/goods_pk-0000002349269256)**
- **[首页下拉进入二楼](https://developer.huawei.com/consumer/cn/doc/architecture-guides/second_floor-0000002386145197)**
- **[地址列表与地址管理](https://developer.huawei.com/consumer/cn/doc/architecture-guides/address_manager-0000002366628616)**
```

The two pages are the same kind of object - a navigation node with no code -
but only one of them still points anywhere useful. If you are looking for the
industry's actual index, it is `SHOP-02`, not this.

## Permissions & config

None. No sample, no `module.json5`, no permissions.

## Constraints

- **Verification level is low, and lower than any other card in this pack.**
  There is no zip, so nothing here is compile-verified. What was checked is
  that the crawled document is complete and that its single link is
  well-formed; the destination portal's contents were not reviewed and are
  outside the scope of this review.
- The page has no 约束与限制 (constraints), 工程目录 (project tree) or
  代码下载 (code download) section, unlike every scenario document in the
  industry. Tooling that assumes those sections exist will find nothing.
- The FAQ destination is a general HarmonyOS phone FAQ. It is not
  shopping-specific and its contents can change independently of this tree, so
  do not cite it as an authority for anything scenario-level.
- This node's content may return: the page still occupies its slot in the
  architecture guide's sidebar, so a future crawl could find it repopulated.
  If it does, this card needs rewriting rather than extending.

## Pitfalls

- **No defects found.** The page is short, its single link is well-formed, and
  the crawl captured it intact. There is no snippet to be mangled, so the
  corpus-wide excerpt defect (`HW-16-0013`) does not apply here - though it
  applies to most of the documents this card routes to, which is why every
  scenario card quotes the zip rather than the document.
- The trap is spending time here. An empty node in a sidebar tree looks like a
  crawl failure; it is not. If a tool or a reviewer reports "SHOP-24 has no
  content", that report is correct and expected.

## References

- `huawei_industry_tree/16_shopping/docs/24_practice-purchase-app-architecture-v1_2.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-purchase-app-architecture-v1_2-0000002298474325
- `SHOP-01` - the industry architecture narrative and the only multi-module sample
- `SHOP-02` - the scenario index this node is often confused with
- `SHOP-21`, `SHOP-22`, `SHOP-23` - the last three scenarios in that index, all reviewed against their zips
