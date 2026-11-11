# ADR-0157：JavDB 接口答不了时刮削退回详情页

- 状态：已采纳
- 日期：2026-11-12
- 关系：修订 [ADR-0144](0144-javdb-scraping-uses-the-anonymous-api.md) 第一条「详情页解析只留给视频预览」；接口的间隔、预算、按主机冷却（[ADR-0148](0148-javdb-api-host-and-signature-fallback.md)）与快照复用不变

## 背景

2026-11-10 起，匿名接口三家主机（`jdforrepam.com`、`apidd.czssdgz.com`、`apidd.spthgb.com`）对搜索 `/v2/search`
与详情 `/v4/movies/<id>` 都回 HTTP 200、`{"success":0,"action":"JWTVerificationError","message":"請登錄帳號"}`。
网页签名与 App 签名两种请求形态结果相同。网页前端 `catembylegacy.fastcdn.dpdns.org` 同期对所有出口回 Cloudflare
403（日本、台湾、香港、新加坡、韩国、荷兰、美国的机房与家宽出口，以及 curl_cffi 的浏览器指纹都试过）。
Wayback 存档里 9～10 月的前端脚本，按 `public_signature_parts` 读出的签名常量与 `APP_SIGNATURE` 相同，所以问题不在签名。

刮削只走接口，接口一停，所有番号的 JavDB 一档都记作「来源暂时不可用」，有码链上的资料、标签与封面印证一起缺。
同期 javdb.com 搜索页与详情页不带 Cookie 仍可读，详情页给出 `JavDBSource.parse` 用到的全部面板字段。在一台部署上实测
IPX-125、SSIS-308、IPX-259 都取得原标题、女优与演员 id、片商、发行、系列、导演、日期、片长、类别和封面；
番号不在站上的作品照常归 `not_found`。

## 决策

**一、接口冷却或未取得时改读详情页。** `JavDBPublicSource.query` 先按 ADR-0144 问接口；`PublicSourceUnavailable`
（包括冷却 `PublicSourcePaused`）时，同一问交给 `JavDBSource(JAVDB)`：搜索页找唯一的 `/v/<id>`，再读详情页。
解析器名是 `javdb-page`，候选与账本证据分得清取法。

**二、接口给出的结论仍按接口算。** 搜索无命中（`not_found`）、同番号多个作品（`identity_ambiguous`）、超出每问预算
（`timeout`）不改读详情页：这几种是接口答复过的结论，换取法不会更准，只会多占 javdb.com 的配额。

**三、详情页那一问走 javdb.com 的规矩。** 主机间隔、按出口 IP 的来源冷却与登录墙判断都沿用 `JAVDB` 与
`JavDBSource` 原有的设置（每页 3 秒，一部片两页）。登录页仍归 `auth_required`，撞上限流或封禁时冷却记在 `javdb` 名下。

## 影响

- 接口恢复后行为与 ADR-0144 相同，详情页只在接口答不了时被问到。
- 接口停用期间每部片多两页 javdb.com 请求；按每页 3 秒，一批 100 部约 10 分钟，仍低于 ADR-0144 记的每分钟 40～50 页封禁线。
- 详情页标题取 `origin-title`，没有时取 `current-title`，后者可能带番号前缀（实测 IPX-259 即是）；与接口给的原标题不同时按来源策略结算。
- 详情页给的封面在 `jdbstatic.com`，与接口给的 `tp.spfcas.com` 主机不同；两者都只作印证证据，不直接用作封面。
